---
layout:     post
title:      Spark源码剖析_Master机制详解（基于2.2.0版本）
subtitle:   主备切换，注册，状态改变，资源调度
date:       2020-02-12
author:     walker
header-img: img/scala-1.jpg
catalog: true
tags:
    - Spark
    - 源码
---

# Spark源码剖析_Master机制详解（基于2.2.0版本）



## Master主备切换机制

1. Master可以配置两个，Spark的standalone模式支持主备切换。

2. Master主备切换可以基于两种模式，一种是基于文件系统的，另一种是基于Zookeeper的：

   - 基于文件系统的，需要在Active Master挂掉之后，手动切换到Standby Master上。
   - 基于Zookeeper的，可以实现自动切换Master。

3. 切换流程（Standby Master角度）：

   1. 使用持久化引擎（FileSystemPersistenceEngine，ZooKeeperPersistenceEngine）去读取storedApps，storedDrivers，storedWorkers信息。
   2. 判断，如果storedApps，storedDrivers，storedWorkers有任意一个信息非空，则继续。
   3. 将持久化的Application，Driver，Worker的信息重新注册，注册到Master内部的内存缓存结构中。
   4. 将Application和Worker的状态都改成UNKOWN，然后向Application所对应的Driver，Worker发送Standby Master的地址。
   5. Driver和Worker如果目前都是正常运行的状态，那么在接收到Standby Master发过来的地址之后，就会返回消息给Standby Master。
   6. 此时，Master陆续接收到Driver和Worker发送过来的响应消息之后，会使用completeRecovery()方法对没有发送响应消息的Driver和Worker进行清理，过滤掉它们的信息。
   7. 调用Standby Master自己的schedule()方法，对正在等待资源调度的Driver和Application进行调度，比如在某个Worker上启动Driver，或为某个Application在Worker上启动Executor。

4. completeRecovery方法解析：

   ```scala
   private def completeRecovery() {
       // Ensure "only-once" recovery semantics using a short synchronization period.
       if (state != RecoveryState.RECOVERING) { return }
       state = RecoveryState.COMPLETING_RECOVERY
       
       // Kill off any workers and apps that didn't respond to us.
       // 未回应的worker，认为已经挂掉了，从内存结构和持久化引擎中删掉
       workers.filter(_.state == WorkerState.UNKNOWN).foreach(removeWorker)
       // 未回应的application，从内存结构和持久化引擎中删掉，并且标记为finished状态
       // 同时把该application对应的所有的executor都kill掉，从Master的信息中删除
       apps.filter(_.state == ApplicationState.UNKNOWN).foreach(finishApplication)
   
       // Update the state of recovered apps to RUNNING
       apps.filter(_.state == ApplicationState.WAITING).foreach(_.state = ApplicationState.RUNNING)
   
       // Reschedule drivers which were not claimed by any workers
       // 上边的worker清理工作，如果把driver所在worker清理掉了，此时有两种策略，要么重新发布driver，
       // 要么把driver的状态标记为ERROR，然后移除driver
       drivers.filter(_.worker.isEmpty).foreach { d =>
         logWarning(s"Driver ${d.id} was not found after master recovery")
         // 如果driver设置为supervise，那么会Spark会尝试重新发布driver
         if (d.desc.supervise) {
           logWarning(s"Re-launching ${d.id}")
           relaunchDriver(d)
         } else {
           removeDriver(d.id, DriverState.ERROR, None)
           logWarning(s"Did not re-launch ${d.id} because it was not supervised")
         }
       }
   
       state = RecoveryState.ALIVE
       // 重新调度资源，后面会介绍
       schedule()
       logInfo("Recovery complete - resuming operations!")
     }
   ```

## Master注册机制

 1. Worker的注册：

    - Worker在启动之后，就会主动向Master注册。
    - Master过滤，将状态为DEAD的Worker过滤掉，将状态为UNKOWN的Worker，清理掉旧的Worker信息，并注册新的Worker信息。
    - 把Worker加入到内存缓存中。
    - 用持久化引擎把Worker信息进行持久化。
    - 调用schedule()方法。
2. Driver的注册：

    - 用spark-submit提交application时，会向Master注册。
    - 将Driver信息加入到内存缓存中。
    - 加入等待调度队列（ArrayBuffer）。
    - 用持久化引擎把Driver信息进行持久化。
    - 调用schedule()方法。

 3. Application的注册：

    - Driver启动好了之后，开始执行我们编写的代码，初始化SparkContext，底层的SchedulerBackend会创建一个AppClient，此AppClient的身份作为一个rpc的端点，会注册到监听器上，携带Application信息和Master的地址，向Master发送注册请求。

    - 将Application信息放入内存缓存中。

    - 将Application加入等待调度队列（ArrayBuffer）。

    - 用持久化引擎把Application信息进行持久化。

    - 调用schedule()方法。

    - 源码分析：

      ```scala
      RegisterApplication(description, driver) =>
            // 如果当前master是Standby Master，不做处理
            if (state == RecoveryState.STANDBY) {
              // ignore, don't send response
            } else {
              logInfo("Registering app " + description.name)
              // 创建一个ApplicationInfo对象
              val app = createApplication(description, driver)
              // 此方法中，回家Application的各种信息加入到Master的内存缓存中
              // 同时将此Application加入到等待调度的队列中
              registerApplication(app)
              logInfo("Registered app " + description.name + " with ID " + app.id)
              // 持久化引擎将Application信息加入到持久化的信息中
              persistenceEngine.addApplication(app)
              // 向发送注册请求的AppClient发送消息，通知对方注册成功
              driver.send(RegisteredApplication(app.id, self))
              // 进行调度
              schedule()
            }
      ```

## Master状态改变处理机制

 1. 如果Driver的状态是错误、完成、杀掉或者失败，那么就移除此Driver，具体实现为removeDriver()方法：

    ```scala
    private def removeDriver(
          driverId: String,
          finalState: DriverState,
          exception: Option[Exception]) {
        drivers.find(d => d.id == driverId) match {
          // 根据DriverId查找Driver，如果找到了，那么从内存缓存中移除Driver信息
          case Some(driver) =>
            logInfo(s"Removing driver: $driverId")
            drivers -= driver
            if (completedDrivers.size >= RETAINED_DRIVERS) {
              val toRemove = math.max(RETAINED_DRIVERS / 10, 1)
              completedDrivers.trimStart(toRemove)
            }
            completedDrivers += driver
            // 把此Driver信息从持久化存储引擎中移除
            persistenceEngine.removeDriver(driver)
            // 更新Driver的最终状态和异常信息
            driver.state = finalState
            driver.exception = exception
            // 从Driver所在的Worker中移除此Driver的缓存信息
            driver.worker.foreach(w => w.removeDriver(driver))
            // 重新进行调度
            schedule()
          case None =>
            logWarning(s"Asked to remove unknown driver: $driverId")
        }
      }
    
    ```

    Executor的状态更新机制跟Driver类似，源码如下：

    ```scala
    case ExecutorStateChanged(appId, execId, state, message, exitStatus) =>
          val execOption = idToApp.get(appId).flatMap(app => app.executors.get(execId))
          execOption match {
            case Some(exec) =>
              // 获取Executor对应Application的信息
              val appInfo = idToApp(appId)
              val oldState = exec.state
              // 更新Executor的状态
              exec.state = state
    
              if (state == ExecutorState.RUNNING) {
                // 状态机只能是LAUNCHING到RUNNING
                assert(oldState == ExecutorState.LAUNCHING,
                  s"executor $execId state transfer from $oldState to RUNNING is illegal")
                // 重置重试次数
                appInfo.resetRetryCount()
              }
    
              // 通知Driver，Executor的状态已经更新
              exec.application.driver.send(ExecutorUpdated(execId, state, message, exitStatus, false))
    
              if (ExecutorState.isFinished(state)) {
                // Remove this executor from the worker and app
                logInfo(s"Removing executor ${exec.fullId} because it is $state")
                // If an application has already finished, preserve its
                // state to display its information properly on the UI
                if (!appInfo.isFinished) {
                  appInfo.removeExecutor(exec)
                }
                exec.worker.removeExecutor(exec)
    
                val normalExit = exitStatus == Some(0)
                // Only retry certain number of times so we don't go into an infinite loop.
                // Important note: this code path is not exercised by tests, so be very careful when
                // changing this `if` condition.
                // 如果是非正常退出，并且该Executor对应的Application中的所有Executor都没有在运行的状态，
                // 同时重试次数也已经超过了 MAX_EXECUTOR_RETRIES（默认10次）
                // 那么认为该Application就是失败了，从各种存储结构中移除该Application信息
                if (!normalExit
                    && appInfo.incrementRetryCount() >= MAX_EXECUTOR_RETRIES
                    && MAX_EXECUTOR_RETRIES >= 0) { // < 0 disables this application-killing path
                  val execs = appInfo.executors.values
                  if (!execs.exists(_.state == ExecutorState.RUNNING)) {
                    logError(s"Application ${appInfo.desc.name} with ID ${appInfo.id} failed " +
                      s"${appInfo.retryCount} times; removing it")
                    removeApplication(appInfo, ApplicationState.FAILED)
                  }
                }
              }
              // 重新进行调度
              schedule()
            case None =>
              logWarning(s"Got status update for unknown executor $appId/$execId")
          }
    ```

## Master资源调度机制

 1. Master的资源调度算法的主要实现就是在前面多次提到的schedule()方法中，源码如下：

    ```scala
    private def schedule(): Unit = {
        // 如果Master的状态不是ALIVE，那么直接返回
        // Standby Master是不会进行资源调度的
        if (state != RecoveryState.ALIVE) {
          return
        }
        // Random.shuffle将集群的Worker列表随机的打乱
        // 取出了workers中之前所有已经注册上来并且状态为ALIVE的worker
        val shuffledAliveWorkers = Random.shuffle(workers.toSeq.filter(_.state == WorkerState.ALIVE))
        val numWorkersAlive = shuffledAliveWorkers.size
        var curPos = 0
        // 调度Driver
        // 只有在yarn-cluster这种模式中才会调度Driver，因为在standalone和yarn-client模式下，Driver都是在	// 提交代码的机器本地启动Driver，不需要调度
        
        // 遍历等待调度的Driver
        for (driver <- waitingDrivers.toList) { // iterate over a copy of waitingDrivers
          // We assign workers to each waiting driver in a round-robin fashion. For each driver, we
          // start from the last worker that was assigned a driver, and continue onwards until we have
          // explored all alive workers.
          var launched = false
          var numWorkersVisited = 0
          // 只要还有没遍历到的worker，并且Driver还没有启动，那么就接着遍历
          while (numWorkersVisited < numWorkersAlive && !launched) {
            // 取出当前遍历到的worker信息
            val worker = shuffledAliveWorkers(curPos)
            numWorkersVisited += 1
            // 如果该worker的cpu数和内存大小都能够满足driver的要求
            if (worker.memoryFree >= driver.desc.mem && worker.coresFree >= driver.desc.cores) {
              // 在该worker上启动Driver
              launchDriver(worker, driver)
              // 把该Driver从等待调度的存储结构中移除
              waitingDrivers -= driver
              // 把启动标志置为true
              launched = true
            }
            // 如果该worker不满足要求，那么把指针指向集合中的下一个worker
            curPos = (curPos + 1) % numWorkersAlive
          }
        }
        // 在worker上分配Executor
        startExecutorsOnWorkers()
      }
    ```

    在上边的代码中我们看到在worker上启动Driver的方法是launchDriver()，起代码如下

    ```scala
    private def launchDriver(worker: WorkerInfo, driver: DriverInfo) {
        logInfo("Launching driver " + driver.id + " on worker " + worker.id)
        // 将Driver信息加入到Worker的内存缓存结构中
        // 将worker内的cpu和内存数量，都加上Driver的cpu和内存数量
        worker.addDriver(driver)
        // 设置该Driver对应的worker信息
        driver.worker = Some(worker)
        // 发送消息给worker，告诉它启动Driver
        worker.endpoint.send(LaunchDriver(driver.id, driver.desc))
        // 设置Driver状态
        driver.state = DriverState.RUNNING
      }
    ```

    在启动Driver之后，调度机制会接着在worker上给Application分配Executor，使用的是startExecutorsOnWorkers()方法，源码如下：

    ```scala
    private def startExecutorsOnWorkers(): Unit = {
        // 首先遍历等待调度的Application，并且过滤出其中依旧需要cpu的
        for (app <- waitingApps if app.coresLeft > 0) {
          val coresPerExecutor: Option[Int] = app.desc.coresPerExecutor
          // Filter out workers that don't have enough resources to launch an executor
          // 过滤出符合条件的worker集合，条件如下：
          // 1.worker状态为ALIVE
          // 2.worker剩余内存大于等于Application每个Executor需要的内存
          // 3.worker剩余cpu数大于等于Application每个Executor需要的cpu数
          // 4.按照cpu数倒序排序
          val usableWorkers = workers.toArray.filter(_.state == WorkerState.ALIVE)
            .filter(worker => worker.memoryFree >= app.desc.memoryPerExecutorMB &&
              worker.coresFree >= coresPerExecutor.getOrElse(1))
            .sortBy(_.coresFree).reverse
          // 在可用worker集合上调度Executor
          // spreadOutApps参数是spark.deploy.spreadOut配置项的值，用户可以指定，默认为true
          // 当值为true时，表示每个Application尽可能在所有节点中分散地分配Executor
          // assignedCores表示要在每个worker上分配的cpu数集合
          val assignedCores = scheduleExecutorsOnWorkers(app, usableWorkers, spreadOutApps)
    
          // Now that we've decided how many cores to allocate on each worker, let's allocate them
          for (pos <- 0 until usableWorkers.length if assignedCores(pos) > 0) {
            // 在worker上分配资源给Executor
            allocateWorkerResourceToExecutors(
              app, assignedCores(pos), coresPerExecutor, usableWorkers(pos))
          }
        }
      }
    ```

    scheduleExecutorsOnWorkers方法源码如下：

    ```scala
    private def scheduleExecutorsOnWorkers(
          app: ApplicationInfo,
          usableWorkers: Array[WorkerInfo],
          spreadOutApps: Boolean): Array[Int] = {
        val coresPerExecutor = app.desc.coresPerExecutor
        val minCoresPerExecutor = coresPerExecutor.getOrElse(1)
        // 如果没有指定每个Executor多个核，那么一个worker上就一个Executor
        val oneExecutorPerWorker = coresPerExecutor.isEmpty
        val memoryPerExecutor = app.desc.memoryPerExecutorMB
        val numUsable = usableWorkers.length
        val assignedCores = new Array[Int](numUsable) // 每个worker分配的cpu数
        val assignedExecutors = new Array[Int](numUsable) // 每个worker上的Executor数
        // 获取要分配多少cpu，取Application剩余需要分配的cpu和worker集合中剩余cpu总和中的最小值
        var coresToAssign = math.min(app.coresLeft, usableWorkers.map(_.coresFree).sum)
        
        // 此处是canLaunchExecutor()方法定义，移至下面
    
        // 过滤出可以启动Executor的worker集合，过滤条件canLaunchExecutor()方法下面会分析
        var freeWorkers = (0 until numUsable).filter(canLaunchExecutor)
        while (freeWorkers.nonEmpty) {
          // 遍历可启动Executor的worker集合
          freeWorkers.foreach { pos =>
            var keepScheduling = true
            while (keepScheduling && canLaunchExecutor(pos)) {
              // 代码进入到这里，即表示可以分配cpu了，先把需要分配的总cpu数减去一个Executor需要的的cpu数
              coresToAssign -= minCoresPerExecutor
              // 把当前worker分配的cpu数累加
              assignedCores(pos) += minCoresPerExecutor
    
              // 如果策略是一个worker就一个Executor，那么把集合当前位置置为1，否则+=1，记录总Executor数
              if (oneExecutorPerWorker) {
                assignedExecutors(pos) = 1
              } else {
                assignedExecutors(pos) += 1
              }
    
              // 如果设置为spreadOut模式，那么就一个worker一个Executor，尽量分散。
              // 此时内层的while循环退出，进入foreach的下一个worker
              // 如果非spreadOut模式，那就先可着一个worker分配，内存while循环继续，资源都分配没了再分配			  // 下一个worker
              if (spreadOutApps) {
                keepScheduling = false
              }
            }
          }
          // 折腾完一个worker之后先把这个worker给滤出去
          freeWorkers = freeWorkers.filter(canLaunchExecutor)
        }
        // 返回每个worker分配的cpu数
        assignedCores
      }
    ```

    ```scala
    	// 判断worker集合中pos位置上的worker资源能否支撑启动Executor
        def canLaunchExecutor(pos: Int): Boolean = {
          // keepScheduling和enoughCores标志位都是用于判断worker能否支撑启动Executor的
          // 如果能分配的cpu数大于每个Executor最小cpu数，keepScheduling标志位置为true
          val keepScheduling = coresToAssign >= minCoresPerExecutor
          // 如果worker的剩余cpu数减去该worker已分配的cpu数后仍大于等于Executor所需cpu数，
          // 那么enoughCores标志位置为true
          val enoughCores = usableWorkers(pos).coresFree - assignedCores(pos) >= minCoresPerExecutor
    
          // If we allow multiple executors per worker, then we can always launch new executors.
          // Otherwise, if there is already an executor on this worker, just give it more cores.
          // 策略相关：
          // 如果一个worker上允许有多个Executor或者当前worker上还未有Executor，
          // launchingNewExecuto标志位置为true
          val launchingNewExecutor = !oneExecutorPerWorker || assignedExecutors(pos) == 0
          if (launchingNewExecutor) {
            // 简要说明：这里是判断worker的内存是否足够，或者是否达到了用户所要求分配的Executor数
            val assignedMemory = assignedExecutors(pos) * memoryPerExecutor
            val enoughMemory = usableWorkers(pos).memoryFree - assignedMemory >= memoryPerExecutor
            val underLimit = assignedExecutors.sum + app.executors.size < app.executorLimit
            keepScheduling && enoughCores && enoughMemory && underLimit
          } else {
            // We're adding cores to an existing executor, so no need
            // to check memory and executor limits
            keepScheduling && enoughCores
          }
        }
    ```

    worker分配资源的allocateWorkerResourceToExecutors方法源码如下：

    ```scala
    private def allocateWorkerResourceToExecutors(
        app: ApplicationInfo,
        assignedCores: Int,
        coresPerExecutor: Option[Int],
        worker: WorkerInfo): Unit = {
      // If the number of cores per executor is specified, we divide the cores assigned
      // to this worker evenly among the executors with no remainder.
      // Otherwise, we launch a single executor that grabs all the assignedCores on this worker.
      // 当前worker上的Executor数
      val numExecutors = coresPerExecutor.map { assignedCores / _ }.getOrElse(1)
      // 给worker分配的cpu数
      val coresToAssign = coresPerExecutor.getOrElse(assignedCores)
      // 挨个启动Executor
      for (i <- 1 to numExecutors) {
        // 把Executor信息加入到Application的内存缓存结构中
        val exec = app.addExecutor(worker, coresToAssign)
        // 启动Executor
        launchExecutor(worker, exec)
        // Executor状态设为RUNNING
        app.state = ApplicationState.RUNNING
      }
    }
    ```

    launchExecutor方法源码如下：

    ```scala
    private def launchExecutor(worker: WorkerInfo, exec: ExecutorDesc): Unit = {
      logInfo("Launching executor " + exec.fullId + " on worker " + worker.id)
      // 把Executor信息加入到worker内存缓存中
      worker.addExecutor(exec)
      // 向worker发送LaunchExecutor消息
      worker.endpoint.send(LaunchExecutor(masterUrl,
        exec.application.id, exec.id, exec.application.desc, exec.cores, exec.memory))
      // 向Application对应的Driver发送ExecutorAdded消息
      exec.application.driver.send(
        ExecutorAdded(exec.id, worker.id, worker.hostPort, exec.cores, exec.memory))
    }
    ```
