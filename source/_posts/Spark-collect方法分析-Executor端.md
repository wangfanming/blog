---
title: Spark collect方法分析-Executor端
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
        // 由Driver上的后端进行TaskSet的分配和启动
        backend.reviveOffers()
      }
    ```

    

