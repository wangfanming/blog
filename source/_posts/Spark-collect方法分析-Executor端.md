---
title: Spark collect方法分析-Executor端
abbrlink: 2768
date: 2023-12-17 15:21:04
tags:
---

## 简介

​	在Spark框架中，理解用户代码是怎么被框架执行的，有助于我们写出高效高质量的代码，也有利于我们在问题分析时可以快速准确的定位问题。

​	collect方法作为一个简单的方法，其执行过程也足以展示整个框架的执行流程。

## 目的

​	通过对调用过程的分析，深入理解整个框架的计算过程，从根本上理解程序的执行逻辑。

<!--more-->

## 流程分析

​	在上文（collect方法的Driver端分析）中，

```scala
taskScheduler.submitTasks(new TaskSet(
      tasks.toArray, stage.id, stage.latestInfo.attemptNumber, jobId, properties))
```

被DAGscheduler构建出来的TaskSet，会由taskScheduler进行调度执行。接下来，我们针对后续流程进行跟踪分析。

- 提交后的流程分析

  - org.apache.spark.scheduler.TaskSchedulerImpl#submitTasks

    ```scala
      override def submitTasks(taskSet: TaskSet) {
        val tasks = taskSet.tasks
        logInfo("Adding task set " + taskSet.id + " with " + tasks.length + " tasks")
        this.synchronized {
          // 组合模式，创建TaskSetManager，将TaskSet托管给TaskSetManager，也就是说每个TaskSet有一个TaskSetManager
          val manager = createTaskSetManager(taskSet, maxTaskFailures)
          val stage = taskSet.stageId
          val stageTaskSets =
            taskSetsByStageIdAndAttempt.getOrElseUpdate(stage, new HashMap[Int, TaskSetManager])
    
          // Mark all the existing TaskSetManagers of this stage as zombie, as we are adding a new one.
          // This is necessary to handle a corner case. Let's say a stage has 10 partitions and has 2
          // TaskSetManagers: TSM1(zombie) and TSM2(active). TSM1 has a running task for partition 10
          // and it completes. TSM2 finishes tasks for partition 1-9, and thinks he is still active
          // because partition 10 is not completed yet. However, DAGScheduler gets task completion
          // events for all the 10 partitions and thinks the stage is finished. If it's a shuffle stage
          // and somehow it has missing map outputs, then DAGScheduler will resubmit it and create a
          // TSM3 for it. As a stage can't have more than one active task set managers, we must mark
          // TSM2 as zombie (it actually is).
          stageTaskSets.foreach { case (_, ts) =>
            ts.isZombie = true
          }
          stageTaskSets(taskSet.stageAttemptId) = manager
          // schedulableBuilder 会在org.apache.spark.scheduler.TaskSchedulerImpl.initialize()时进行初始化，
          // 该配置由“spark.scheduler.mode”的具体配置决定，默认FIFO调度策略
          // 根据配置，采用不同的调度策略执行TaskSet
          schedulableBuilder.addTaskSetManager(manager, manager.taskSet.properties)
    
          if (!isLocal && !hasReceivedTask) {
            starvationTimer.scheduleAtFixedRate(new TimerTask() {
              override def run() {
                if (!hasLaunchedTask) {
                  logWarning("Initial job has not accepted any resources; " +
                    "check your cluster UI to ensure that workers are registered " +
                    "and have sufficient resources")
                } else {
                  this.cancel()
                }
              }
            }, STARVATION_TIMEOUT_MS, STARVATION_TIMEOUT_MS)
          }
          hasReceivedTask = true
        }
        // backend:在org.apache.spark.SparkContext.createTaskScheduler()时，根据具体配置，会生成相应的后端，
        // 这些后端都是CoarseGrainedSchedulerBackend的子接口的YarnSchedulerBackend的实现
        // 这根具体的部署模式（cluser,client）相关,然后由相应的后端程序开始制作票据，交由Executor进行TaskSet的处理
        backend.reviveOffers()
      }
    ```
  
  - 在org.apache.spark.scheduler.cluster.CoarseGrainedSchedulerBackend#reviveOffers方法中，由driver发出ReviveOffers事件：
  
    ```scala
    override def reviveOffers() {
        //向DriverEndpoint发送ReviveOffers事件
        driverEndpoint.send(ReviveOffers)
      }
    ```
  
  - 该事件被org.apache.spark.scheduler.cluster.CoarseGrainedSchedulerBackend.DriverEndpoint#receive接收后，所有已经在yarn上启动的Executor开始制作票据
  
    ```scala
    private def makeOffers() {
          // Make sure no executor is killed while some task is launching on it
          val taskDescs = withLock {
            // Filter out executors under killing
            val activeExecutors = executorDataMap.filterKeys(executorIsAlive)
            val workOffers = activeExecutors.map {
              case (id, executorData) =>
                new WorkerOffer(id, executorData.executorHost, executorData.freeCores,
                  Some(executorData.executorAddress.hostPort))
            }.toIndexedSeq
            // 把可以执行任务的Executors告诉TaskScheduler，开始执行Task
            scheduler.resourceOffers(workOffers)
          }
          if (!taskDescs.isEmpty) {
            // 向Executor发出执行Task的事件
            launchTasks(taskDescs)
          }
        }			
    ```
  
  - 在票据制作过程中，检测所有正常的Executor,并将TaskSet分配给他们进行执行
  
    ```scala
    private def launchTasks(tasks: Seq[Seq[TaskDescription]]) {
          for (task <- tasks.flatten) {
            val serializedTask = TaskDescription.encode(task)
            if (serializedTask.limit() >= maxRpcMessageSize) {
              Option(scheduler.taskIdToTaskSetManager.get(task.taskId)).foreach { taskSetMgr =>
                try {
                  var msg = "Serialized task %s:%d was %d bytes, which exceeds max allowed: " +
                    "spark.rpc.message.maxSize (%d bytes). Consider increasing " +
                    "spark.rpc.message.maxSize or using broadcast variables for large values."
                  msg = msg.format(task.taskId, task.index, serializedTask.limit(), maxRpcMessageSize)
                  taskSetMgr.abort(msg)
                } catch {
                  case e: Exception => logError("Exception in error callback", e)
                }
              }
            }
            else {
              //executorDataMap在Driver->SparkSession->SparkContext->SparkEnv初始化的时候创建，里面存储了每个Executor的信息
              //task中包含有分区的最佳位置信息，根据这个信息，找到最佳的Executor
              val executorData = executorDataMap(task.executorId)
              executorData.freeCores -= scheduler.CPUS_PER_TASK
    
              logDebug(s"Launching task ${task.taskId} on executor id: ${task.executorId} hostname: " +
                s"${executorData.executorHost}.")
              // 使用选定的Executor的executorEndpoint发出LaunchTask，在这个Executor上启动task的处理
              executorData.executorEndpoint.send(LaunchTask(new SerializableBuffer(serializedTask)))
            }
          }
        }
    ```
  
  - 在Executor的org.apache.spark.executor.CoarseGrainedExecutorBackend#receive中，LaunchTask事件会被处理。
  
    ```scala
     override def receive: PartialFunction[Any, Unit] = {
        case RegisteredExecutor =>
          logInfo("Successfully registered with driver")
          try {
          //向Driver注册成功后,创建Executor
            executor = new Executor(executorId, hostname, env, userClassPath, isLocal = false)
          } catch {
            case NonFatal(e) =>
              exitExecutor(1, "Unable to create executor due to " + e.getMessage, e)
          }
        // 接收LaunchTask中的TaskSet,进行处理
        case LaunchTask(data) =>
          if (executor == null) {
            exitExecutor(1, "Received LaunchTask command but executor was null")
          } else {
            val taskDesc = TaskDescription.decode(data.value)
            logInfo("Got assigned task " + taskDesc.taskId)
            executor.launchTask(this, taskDesc)
          }
    ```
  
  - 在org.apache.spark.executor.Executor#launchTask中，会将TaskSet放入线程池进行处理
  
    ```scala
    def launchTask(context: ExecutorBackend, taskDescription: TaskDescription): Unit = {
        val tr = new TaskRunner(context, taskDescription)
        runningTasks.put(taskDescription.taskId, tr)
        threadPool.execute(tr)
      }
    ```
  
  - 在org.apache.spark.executor.Executor.TaskRunner#run中
  
    1. 会对taskDescription进行反序列化为Task对象，具体的计算过程在org.apache.spark.scheduler.Task#run完成。
  
       - 在org.apache.spark.scheduler.Task#run中，生成任务的上下文对象，并在org.apache.spark.scheduler.Task#runTask中进行计算。
  
         ```scala
           final def run(
               taskAttemptId: Long,
               attemptNumber: Int,
               metricsSystem: MetricsSystem): T = {
             SparkEnv.get.blockManager.registerTask(taskAttemptId)
             // TODO SPARK-24874 Allow create BarrierTaskContext based on partitions, instead of whether
             // the stage is barrier.
             val taskContext = new TaskContextImpl(
               stageId,
               stageAttemptId, // stageAttemptId and stageAttemptNumber are semantically equal
               partitionId,
               taskAttemptId,
               attemptNumber,
               taskMemoryManager,
               localProperties,
               metricsSystem,
               metrics)
         ....
             new CallerContext(
               "TASK",
               SparkEnv.get.conf.get(APP_CALLER_CONTEXT),
               appId,
               appAttemptId,
               jobId,
               Option(stageId),
               Option(stageAttemptId),
               Option(taskAttemptId),
               Option(attemptNumber)).setCurrentContext()
         
             try {
               runTask(context)
             } catch { ...
         ```
  
         Task根据Stage种类，分为org.apache.spark.scheduler.ResultTask和org.apache.spark.scheduler.ShuffleMapTask。
  
         - 在org.apache.spark.scheduler.ShuffleMapTask#runTask中
  
         ```scala
         override def runTask(context: TaskContext): MapStatus = {
             ...
             val ser = SparkEnv.get.closureSerializer.newInstance()
             val (rdd, dep) = ser.deserialize[(RDD[_], ShuffleDependency[_, _, _])](
               ByteBuffer.wrap(taskBinary.value), Thread.currentThread.getContextClassLoader)
         else 0L
         ...
             var writer: ShuffleWriter[Any, Any] = null
             try {
               val manager = SparkEnv.get.shuffleManager
               writer = manager.getWriter[Any, Any](dep.shuffleHandle, partitionId, context)
               //按照rdd的转换函数，将rdd的数据转换为Iterator[Product2[K, V]]，并将结果写入writer
               writer.write(rdd.iterator(partition, context).asInstanceOf[Iterator[_ <: Product2[Any, Any]]])
               writer.stop(success = true).get
             } catch {
               ...
             }
           }
         ```
  
         - 在org.apache.spark.scheduler.ResultTask#runTask中：
  
         ```scala
         override def runTask(context: TaskContext): U = {
             // Deserialize the RDD and the func using the broadcast variables.
             val threadMXBean = ManagementFactory.getThreadMXBean
             val deserializeStartTime = System.currentTimeMillis()
             val deserializeStartCpuTime = if (threadMXBean.isCurrentThreadCpuTimeSupported) {
               threadMXBean.getCurrentThreadCpuTime
             } else 0L
             val ser = SparkEnv.get.closureSerializer.newInstance()
             val (rdd, func) = ser.deserialize[(RDD[T], (TaskContext, Iterator[T]) => U)](
               ByteBuffer.wrap(taskBinary.value), Thread.currentThread.getContextClassLoader)
             _executorDeserializeTime = System.currentTimeMillis() - deserializeStartTime
             _executorDeserializeCpuTime = if (threadMXBean.isCurrentThreadCpuTimeSupported) {
               threadMXBean.getCurrentThreadCpuTime - deserializeStartCpuTime
             } else 0L
         	//使用用户定义的函数func对RDD的partition进行计算
             func(context, rdd.iterator(partition, context))
           }
         ```
  
         该func方法在org.apache.spark.rdd.RDD#collect()就是：
  
         ```scala
         (iter: Iterator[T]) => iter.toArray	
         ```
  
         - 最后在Driver端：
  
         ```scala
         Array.concat(results: _*)
         ```
  
         会将各个分区的数组拼在一起作为最终结果。
  
    2. 将计算结果进行序列化，如果结果超过了maxDirectResultSize限制，还会生成Block存储，并将信息以StatusUpdate事件通报给Driver。
  
       - 在org.apache.spark.scheduler.cluster.CoarseGrainedSchedulerBackend.DriverEndpoint#receive中，接收到StatusUpdate后即可对后续的TaskSet进行展开。通过计算各个任务的本地性，依据最佳本地性按顺序进行任务调度。
  
         ```scala
          override def receive: PartialFunction[Any, Unit] = {
               case StatusUpdate(executorId, taskId, state, data) =>
                 scheduler.statusUpdate(taskId, state, data.value)
                 if (TaskState.isFinished(state)) {
                   executorDataMap.get(executorId) match {
                     case Some(executorInfo) =>
                       executorInfo.freeCores += scheduler.CPUS_PER_TASK
                       //根据已经完成的Task,根据其所在Executor的ID，通过计算任务本地性，重新调度任务
                       makeOffers(executorId)
                     case None =>
                       // Ignoring the update since we don't know about the executor.
                       logWarning(s"Ignored task status update ($taskId state $state) " +
                         s"from unknown executor with ID $executorId")
                   }
                 }
         ```
  
         
  
    3. 更新度量系统的数据，用于展示任务执行信息。



### 总结

上述大致整理出了Stage划分后，TaskSet在Executor中的调度执行流程。但是其中隐藏了很多细节，如本地性计算过程、调度策略、各个组件是如何初始化、如何进行通信的。具体信息需要查看这个流程中相关方法栈进行分析。

