---
title: Spark collect方法调用分析
categories:
  - 大数据
tags:
  - spark
abbrlink: 13633
date: 2023-12-12 00:54:21
---



通过对调用过程的分析，深入理解整个框架的计算过程，从根本上理解程序的执行逻辑。

<!--more-->


- 方法调用链分析：

  1. 进入org.apache.spark.rdd.RDD#collect()

    ```java
      def collect(): Array[T] = withScope {
        val results = sc.runJob(this, (iter: Iterator[T]) => iter.toArray)
        Array.concat(results: _*)
      }
    ```

    - 上述withScope是一个柯里化操作，方法体内为一个匿名函数，这样做是为了在任务提交前对SparkContext做一些设置。

  2. org.apache.spark.SparkContext#runJob方法：

    ```scala
    def runJob[T, U: ClassTag](rdd: RDD[T], func: Iterator[T] => U): Array[U] = {
        // 默认的分区数等于RDD的分区数。
      runJob(rdd, func, 0 until rdd.partitions.length)
    }
    ```

  3. org.apache.spark.SparkContext#runJob方法经过多次重载调用，

    ```scala
    def runJob[T, U: ClassTag](
        rdd: RDD[T],
        func: (TaskContext, Iterator[T]) => U,
        partitions: Seq[Int]): Array[U] = {
      val results = new Array[U](partitions.size)
      runJob[T, U](rdd, func, partitions, (index, res) => results(index) = res)
      results
    }
    ```

    - (index, res) => results(index) = res 该函数表示将每个分区的计算结果存放在数组results相应的索引上。

    ```scala
    def runJob[T, U: ClassTag](
        rdd: RDD[T],
        func: (TaskContext, Iterator[T]) => U,
        partitions: Seq[Int],
        resultHandler: (Int, U) => Unit): Unit = {
      ...
        //对func提前预设闭包清除
      val cleanedFunc = clean(func)
     ...
        //调用DAGScheduler进行DAG构建
      dagScheduler.runJob(rdd, cleanedFunc, partitions, callSite, resultHandler, localProperties.get)
      progressBar.foreach(_.finishAll())
      // 对需要进行保存的RDD进行保存
      rdd.doCheckpoint()
    }
    ```

  4. org.apache.spark.scheduler.DAGScheduler#runJob

    ```scala
    def runJob[T, U](
        rdd: RDD[T],
        func: (TaskContext, Iterator[T]) => U,
        partitions: Seq[Int],
        callSite: CallSite,
        resultHandler: (Int, U) => Unit,
        properties: Properties): Unit = {
      val start = System.nanoTime
      //提交至消息总线，由DAGScheduler.doOnReceive处理
      val waiter = submitJob(rdd, func, partitions, callSite, resultHandler, properties)
      ThreadUtils.awaitReady(waiter.completionFuture, Duration.Inf)
      // 方法阻塞等待任务执行结束
      waiter.completionFuture.value.get match {
        ...
      }
    }
    ```

  5. org.apache.spark.scheduler.DAGScheduler#submitJob

    ```scala
    def submitJob[T, U](
        rdd: RDD[T],
        func: (TaskContext, Iterator[T]) => U,
        partitions: Seq[Int],
        callSite: CallSite,
        resultHandler: (Int, U) => Unit,
        properties: Properties): JobWaiter[U] = {
        
      val maxPartitions = rdd.partitions.length
    ...
      // 生成全局的JobId
      val jobId = nextJobId.getAndIncrement()
      if (partitions.size == 0) {
        // Return immediately if the job is running 0 tasks
        return new JobWaiter[U](this, jobId, 0, resultHandler)
      }
    
      val func2 = func.asInstanceOf[(TaskContext, Iterator[_]) => _]
      // 分区数等于task数
      val waiter = new JobWaiter(this, jobId, partitions.size, resultHandler)
      // 向总线发出JobSubmitted事件，通过监听器模式，通知给所有监听器
      eventProcessLoop.post(JobSubmitted(
        jobId, rdd, func2, partitions.toArray, callSite, waiter,
        SerializationUtils.clone(properties)))
      waiter
    }
    ```

  6. org.apache.spark.scheduler.DAGScheduler#handleJobSubmitted

    ```scala
      private[scheduler] def handleJobSubmitted(jobId: Int,
          finalRDD: RDD[_],
          func: (TaskContext, Iterator[_]) => _,
          partitions: Array[Int],
          callSite: CallSite,
          listener: JobListener,
          properties: Properties) {
        var finalStage: ResultStage = null
        try {
          // New stage creation may throw an exception if, for example, jobs are run on a
          // HadoopRDD whose underlying HDFS files have been deleted.
          // 从最后一个Stage开始向前创建DAG
          finalStage = createResultStage(finalRDD, func, partitions, jobId, callSite)
        } catch {
          case e: BarrierJobSlotsNumberCheckFailed =>
      ...
            val numCheckFailures = barrierJobIdToNumTasksCheckFailures.compute(jobId,
              new BiFunction[Int, Int, Int] {
                override def apply(key: Int, value: Int): Int = value + 1
              })
            if (numCheckFailures <= maxFailureNumTasksCheck) {
              messageScheduler.schedule(
                new Runnable {
                  // job提交失败，在最大允许次数内进行重新提交
                  override def run(): Unit = eventProcessLoop.post(JobSubmitted(jobId, finalRDD, func,
                    partitions, callSite, listener, properties))
    ...
        }
        // Job submitted, clear internal data.
        barrierJobIdToNumTasksCheckFailures.remove(jobId)
    
        val job = new ActiveJob(jobId, finalStage, callSite, listener, properties)
        clearCacheLocs()
    	...
        val jobSubmissionTime = clock.getTimeMillis()
        jobIdToActiveJob(jobId) = job
        activeJobs += job
        finalStage.setActiveJob(job)
        // 任务提交流程 - 根据jobId获取其对应的所有Stages,通知给消息总线，
        // 在org.apache.spark.status.AppStatusListener.onJobStart方法中处理
        val stageIds = jobIdToStageIds(jobId).toArray
        val stageInfos = stageIds.flatMap(id => stageIdToStage.get(id).map(_.latestInfo))
        // 根据SparkListenerJobStart事件，构建stage的DAG执行图,并刷新至UI
        listenerBus.post(
          SparkListenerJobStart(job.jobId, jobSubmissionTime, stageInfos, properties))
        // 提交Stage开始执行
        submitStage(finalStage)
      }
    ```

    - SparkListenerJobStart事件通过消息总线会通知到org.apache.spark.status.AppStatusListener#onJobStart，进行UI界面展示。

  7. org.apache.spark.scheduler.DAGScheduler#submitStage

    ```scala
    private def submitStage(stage: Stage) {
      val jobId = activeJobForStage(stage)
      if (jobId.isDefined) {
        logDebug(s"submitStage($stage (name=${stage.name};" +
          s"jobs=${stage.jobIds.toSeq.sorted.mkString(",")}))")
        if (!waitingStages(stage) && !runningStages(stage) && !failedStages(stage)) {
          // 通过查找missingStage构建出了所有的Stage,由于Stage从后向前构建的特性，
          // flinaStage的missingStage就是所有的直接依赖的Stage
          val missing = getMissingParentStages(stage).sortBy(_.id)
          logDebug("missing: " + missing)
          // 只有最后一个Stage,即最贴近数据源的Stage才满足missing.isEmpty
          if (missing.isEmpty) {
            logInfo("Submitting " + stage + " (" + stage.rdd + "), which has no missing parents")
            // 开始从最后一个Stage提交Task
            submitMissingTasks(stage, jobId.get)
          } else {
            for (parent <- missing) {
              submitStage(parent)
            }
            waitingStages += stage
          }
        }
      } else {
        abortStage(stage, "No active job for stage " + stage.id, None)
      }
    }
    ```

    - 这是一个尾递归方法，由第一个Stage触发上游依赖的Stage的构建，从而完成DAG的构建。

  8. org.apache.spark.scheduler.DAGScheduler#getMissingParentStages

    ```scala
      private def getMissingParentStages(stage: Stage): List[Stage] = {
        val missing = new HashSet[Stage]
        val visited = new HashSet[RDD[_]]
        // We are manually maintaining a stack here to prevent StackOverflowError
        // caused by recursively visiting
        val waitingForVisit = new ArrayStack[RDD[_]]
        def visit(rdd: RDD[_]) {
          if (!visited(rdd)) {
            visited += rdd
            // 查找缓存中的RDD的分区的TaskLocation,没有就通过BlockManagerMaster进行获取并存储
            val rddHasUncachedPartitions = getCacheLocs(rdd).contains(Nil)
            if (rddHasUncachedPartitions) {
              for (dep <- rdd.dependencies) {
                dep match {
                  case shufDep: ShuffleDependency[_, _, _] =>
                    // 构建ShuffleMapStage，优先使用已经创建过的，复用已经生成的ShuffleMapStage的输出
                    val mapStage = getOrCreateShuffleMapStage(shufDep, stage.firstJobId)
                    if (!mapStage.isAvailable) {
                      missing += mapStage
                    }
                  case narrowDep: NarrowDependency[_] =>
                    waitingForVisit.push(narrowDep.rdd)
                }
              }
            }
          }
        }
        waitingForVisit.push(stage.rdd)
        while (waitingForVisit.nonEmpty) {
          visit(waitingForVisit.pop())
        }
        missing.toList
      }
    ```

  9. org.apache.spark.scheduler.DAGScheduler#submitMissingTasks

    ```scala
    /** Called when stage's parents are available and we can now do its task. */
    private def submitMissingTasks(stage: Stage, jobId: Int) {
      logDebug("submitMissingTasks(" + stage + ")")
    
      // First figure out the indexes of partition ids to compute.
      val partitionsToCompute: Seq[Int] = stage.findMissingPartitions()
    
      // Use the scheduling pool, job group, description, etc. from an ActiveJob associated
      // with this Stage
      val properties = jobIdToActiveJob(jobId).properties
    
      runningStages += stage
      // SparkListenerStageSubmitted should be posted before testing whether tasks are
      // serializable. If tasks are not serializable, a SparkListenerStageCompleted event
      // will be posted, which should always come after a corresponding SparkListenerStageSubmitted
      // event.
      stage match {
        case s: ShuffleMapStage =>
          outputCommitCoordinator.stageStart(stage = s.id, maxPartitionId = s.numPartitions - 1)
        case s: ResultStage =>
          outputCommitCoordinator.stageStart(
            stage = s.id, maxPartitionId = s.rdd.partitions.length - 1)
      }
      val taskIdToLocations: Map[Int, Seq[TaskLocation]] = try {
        stage match {
          case s: ShuffleMapStage =>
            partitionsToCompute.map { id => (id, getPreferredLocs(stage.rdd, id))}.toMap
          case s: ResultStage =>
            partitionsToCompute.map { id =>
              val p = s.partitions(id)
              (id, getPreferredLocs(stage.rdd, p))
            }.toMap
        }
      } catch {
        case NonFatal(e) =>
          stage.makeNewStageAttempt(partitionsToCompute.size)
          listenerBus.post(SparkListenerStageSubmitted(stage.latestInfo, properties))
          abortStage(stage, s"Task creation failed: $e\n${Utils.exceptionString(e)}", Some(e))
          runningStages -= stage
          return
      }
    
      stage.makeNewStageAttempt(partitionsToCompute.size, taskIdToLocations.values.toSeq)
    
      // If there are tasks to execute, record the submission time of the stage. Otherwise,
      // post the even without the submission time, which indicates that this stage was
      // skipped.
      if (partitionsToCompute.nonEmpty) {
        stage.latestInfo.submissionTime = Some(clock.getTimeMillis())
      }
      listenerBus.post(SparkListenerStageSubmitted(stage.latestInfo, properties))
    
      // TODO: Maybe we can keep the taskBinary in Stage to avoid serializing it multiple times.
      // Broadcasted binary for the task, used to dispatch tasks to executors. Note that we broadcast
      // the serialized copy of the RDD and for each task we will deserialize it, which means each
      // task gets a different copy of the RDD. This provides stronger isolation between tasks that
      // might modify state of objects referenced in their closures. This is necessary in Hadoop
      // where the JobConf/Configuration object is not thread-safe.
      var taskBinary: Broadcast[Array[Byte]] = null
      var partitions: Array[Partition] = null
      try {
        // For ShuffleMapTask, serialize and broadcast (rdd, shuffleDep).
        // For ResultTask, serialize and broadcast (rdd, func).
        var taskBinaryBytes: Array[Byte] = null
        // taskBinaryBytes and partitions are both effected by the checkpoint status. We need
        // this synchronization in case another concurrent job is checkpointing this RDD, so we get a
        // consistent view of both variables.
        RDDCheckpointData.synchronized {
          taskBinaryBytes = stage match {
            case stage: ShuffleMapStage =>
              JavaUtils.bufferToArray(
                closureSerializer.serialize((stage.rdd, stage.shuffleDep): AnyRef))
            case stage: ResultStage =>
              JavaUtils.bufferToArray(closureSerializer.serialize((stage.rdd, stage.func): AnyRef))
          }
    
          partitions = stage.rdd.partitions
        }
        // 通过广播将闭包序列化后的task信息广播出去
        taskBinary = sc.broadcast(taskBinaryBytes)
      } catch {
        // In the case of a failure during serialization, abort the stage.
        case e: NotSerializableException =>
          abortStage(stage, "Task not serializable: " + e.toString, Some(e))
          runningStages -= stage
    
          // Abort execution
          return
        case e: Throwable =>
          abortStage(stage, s"Task serialization failed: $e\n${Utils.exceptionString(e)}", Some(e))
          runningStages -= stage
    
          // Abort execution
          return
      }
      // 拆解Stage，提取各分区最优计算位置，封装成task集合
      val tasks: Seq[Task[_]] = try {
        val serializedTaskMetrics = closureSerializer.serialize(stage.latestInfo.taskMetrics).array()
        stage match {
          case stage: ShuffleMapStage =>
            stage.pendingPartitions.clear()
            partitionsToCompute.map { id =>
              val locs = taskIdToLocations(id)
              val part = partitions(id)
              stage.pendingPartitions += id
              new ShuffleMapTask(stage.id, stage.latestInfo.attemptNumber,
                taskBinary, part, locs, properties, serializedTaskMetrics, Option(jobId),
                Option(sc.applicationId), sc.applicationAttemptId, stage.rdd.isBarrier())
            }
    
          case stage: ResultStage =>
            partitionsToCompute.map { id =>
              val p: Int = stage.partitions(id)
              val part = partitions(p)
              val locs = taskIdToLocations(id)
              new ResultTask(stage.id, stage.latestInfo.attemptNumber,
                taskBinary, part, locs, id, properties, serializedTaskMetrics,
                Option(jobId), Option(sc.applicationId), sc.applicationAttemptId,
                stage.rdd.isBarrier())
            }
        }
      } catch {
        case NonFatal(e) =>
          abortStage(stage, s"Task creation failed: $e\n${Utils.exceptionString(e)}", Some(e))
          runningStages -= stage
          return
      }
    
      if (tasks.size > 0) {
        logInfo(s"Submitting ${tasks.size} missing tasks from $stage (${stage.rdd}) (first 15 " +
          s"tasks are for partitions ${tasks.take(15).map(_.partitionId)})")
        // 将task集合构建为TaskSet提交给taskScheduler进行计算
        taskScheduler.submitTasks(new TaskSet(
          tasks.toArray, stage.id, stage.latestInfo.attemptNumber, jobId, properties))
      } else {
        // Because we posted SparkListenerStageSubmitted earlier, we should mark
        // the stage as completed here in case there are no tasks to run
        markStageAsFinished(stage, None)
    
        stage match {
          case stage: ShuffleMapStage =>
            logDebug(s"Stage ${stage} is actually done; " +
                s"(available: ${stage.isAvailable}," +
                s"available outputs: ${stage.numAvailableOutputs}," +
                s"partitions: ${stage.numPartitions})")
            markMapStageJobsAsFinished(stage)
          case stage : ResultStage =>
            logDebug(s"Stage ${stage} is actually done; (partitions: ${stage.numPartitions})")
        }
        // 当前Stage计算完成后，开始提交其子Stage进行计算
        submitWaitingChildStages(stage)
      }
    }
    ```

  