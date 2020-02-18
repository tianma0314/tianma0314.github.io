---
layout:     post
title:      Spark源码剖析_Worker详解（基于2.2.0版本）
subtitle:   Driver和Executor的启动
date:       2020-02-15
author:     walker
header-img: img/scala-1.jpg
catalog: true
tags:
    - Spark
    - 源码
---

# Spark源码剖析_Worker详解（基于2.2.0版本）

## Worker启动Driver流程

 1. Worker在接收到LaunchDriver的通知后，会进入到如下代码：

    ```scala
    case LaunchDriver(driverId, driverDesc, resources_) =>
          logInfo(s"Asked to launch driver $driverId")
      	  // Driver启动的逻辑就在DriverRunner中
          val driver = new DriverRunner(
            conf,
            driverId,
            workDir,
            sparkHome,
            driverDesc.copy(command = Worker.maybeUpdateSSLSettings(driverDesc.command, conf)),
            self,
            workerUri,
            securityMgr,
            resources_)
          drivers(driverId) = driver
    	  // 启动DriverRunner
          driver.start()
    
    	  // 记录已使用的cpu资源
          coresUsed += driverDesc.cores
    	  // 记录已使用的内存资源
          memoryUsed += driverDesc.mem
    	  // 记录其他资源使用情况，如GPU等
          addResourcesUsed(resources_)
    ```

    DriverRunner类负责Driver的执行，包括失败的Driver的自动重启，当前只在standalone模式中使用，

    如上述代码显示，DriverRunner的start()方法为启动Driver的核心实现，源码如下：

    ```scala
    private[worker] def start() = {
        // 启动一个java线程去启动Driver
        new Thread("DriverRunner for " + driverId) {
          override def run(): Unit = {
            var shutdownHook: AnyRef = null
            try {
              // 添加关闭的回调钩子，退出时会杀掉进程
              shutdownHook = ShutdownHookManager.addShutdownHook { () =>
                logInfo(s"Worker shutting down, killing driver $driverId")
                kill()
              }
    
              // 准备Driver的jar包，启动Driver，Driver开始运行直至结束或失败或被kill掉
              val exitCode = prepareAndRunDriver()
    
              // set final state depending on if forcibly killed and process exit code
              finalState = if (exitCode == 0) {
                Some(DriverState.FINISHED)
              } else if (killed) {
                Some(DriverState.KILLED)
              } else {
                Some(DriverState.FAILED)
              }
            } catch {
              case e: Exception =>
                kill()
                finalState = Some(DriverState.ERROR)
                finalException = Some(e)
            } finally {
              if (shutdownHook != null) {
                ShutdownHookManager.removeShutdownHook(shutdownHook)
              }
            }
    
            // 无论Driver以何种结果结束运行，线程会把Driver的最终状态发送给worker
            worker.send(DriverStateChanged(driverId, finalState.get, finalException))
          }
        }.start()
      }
    ```

    prepareAndRunDriver()方法源码如下：

    ```scala
    private[worker] def prepareAndRunDriver(): Int = {
        // 创建Driver的工作目录
        val driverDir = createWorkingDirectory()
        // 下载用户的jar包
        val localJarFilename = downloadUserJar(driverDir)
        // 准备资源文件
        val resourceFileOpt = prepareResourcesFile(SPARK_DRIVER_PREFIX, resources, driverDir)
    
        def substituteVariables(argument: String): String = argument match {
          case "{{WORKER_URL}}" => workerUrl
          case "{{USER_JAR}}" => localJarFilename
          case other => other
        }
    
        // config resource file for driver, which would be used to load resources when driver starts up
        val javaOpts = driverDesc.command.javaOpts ++ resourceFileOpt.map(f =>
          Seq(s"-D${DRIVER_RESOURCES_FILE.key}=${f.getAbsolutePath}")).getOrElse(Seq.empty)
        // 创建一个ProcessBuilder，用于启动Driver进程
        val builder = CommandUtils.buildProcessBuilder(driverDesc.command.copy(javaOpts = javaOpts),
          securityManager, driverDesc.mem, sparkHome.getAbsolutePath, substituteVariables)
    	// 启动Driver
        runDriver(builder, driverDir, driverDesc.supervise)
      }
    ```

    runDriver()方法中，主要是重定向了stdout和stderr到文件中，然后调用runCommandWithRetry()方法，runCommandWithRetry()方法源码如下：

    ```scala
    private[worker] def runCommandWithRetry(
          command: ProcessBuilderLike, initialize: Process => Unit, supervise: Boolean): Int = {
        var exitCode = -1
        // Time to wait between submission retries.
        var waitSeconds = 1
        // A run of this many seconds resets the exponential back-off.
        val successfulRunDuration = 5
        var keepTrying = !killed
    
        val redactedCommand = Utils.redactCommandLineArgs(conf, command.command)
          .mkString("\"", "\" \"", "\"")
        while (keepTrying) {
          logInfo("Launch Command: " + redactedCommand)
    
          synchronized {
            if (killed) { return exitCode }
            // 开一个进程，该进程的参数就是spark用户程序的各种参数
            process = Some(command.start())
            initialize(process.get)
          }
    
          val processStart = clock.getTimeMillis()
          // 启动该进程，从而启动Driver
          exitCode = process.get.waitFor()
    
          // 如果supervise为true(spark自管理重试机制)
          // 并且Driver启动失败，并且进程还没被kill掉，那么就重新尝试启动Driver
          keepTrying = supervise && exitCode != 0 && !killed
          if (keepTrying) {
            // 如果Driver启动时间超过5秒，那么就等一秒再启动
            if (clock.getTimeMillis() - processStart > successfulRunDuration * 1000L) {
              waitSeconds = 1
            }
            logInfo(s"Command exited with status $exitCode, re-launching after $waitSeconds s.")
            // 睡waitSeconds秒
            sleeper.sleep(waitSeconds)
            // 重试次数越多，睡眠时间二倍增长
            waitSeconds = waitSeconds * 2 // exponential back-off
          }
        }
    
        exitCode
      }
    ```

    至此，Driver启动流程完毕，就开始运行了，我们应注意到，以上代码是在一个java线程中执行的，当Driver运行完毕后（即用户代码执行完毕），线程会把Driver的状态通知给worker，发送的协议是DriverStateChanged，worker接收DriverStateChanged消息是处理流程如下：

    ```scala
    private[worker] def handleDriverStateChanged(driverStateChanged: DriverStateChanged): Unit = {
        val driverId = driverStateChanged.driverId
        val exception = driverStateChanged.exception
        val state = driverStateChanged.state
        // 根据Driver的最终状态来记录日志
        state match {
          case DriverState.ERROR =>
            logWarning(s"Driver $driverId failed with unrecoverable exception: ${exception.get}")
          case DriverState.FAILED =>
            logWarning(s"Driver $driverId exited with failure")
          case DriverState.FINISHED =>
            logInfo(s"Driver $driverId exited successfully")
          case DriverState.KILLED =>
            logInfo(s"Driver $driverId was killed by user")
          case _ =>
            logDebug(s"Driver $driverId changed state to $state")
        }
        // worker给Master发送DriverStateChanged的消息，Master做对应的处理
        sendToMaster(driverStateChanged)
        // 将Driver移除内存缓存结构
        val driver = drivers.remove(driverId).get
        finishedDrivers(driverId) = driver
        trimFinishedDriversIfNecessary()
        // 释放资源
        memoryUsed -= driver.driverDesc.mem
        coresUsed -= driver.driverDesc.cores
        removeResourcesUsed(driver.resources)
      }
    ```

    在Master接收到worker的DriverStateChanged消息后，也会对该Driver在Master中对应的各种信息做清理，源码如下：

    ```scala
    private def removeDriver(
          driverId: String,
          finalState: DriverState,
          exception: Option[Exception]): Unit = {
        drivers.find(d => d.id == driverId) match {
          case Some(driver) =>
            logInfo(s"Removing driver: $driverId")
            // 将Driver从内存缓存结构中移除
            drivers -= driver
            if (completedDrivers.size >= retainedDrivers) {
              val toRemove = math.max(retainedDrivers / 10, 1)
              completedDrivers.trimStart(toRemove)
            }
            completedDrivers += driver
            // 从持久化引擎中移除该Driver信息
            persistenceEngine.removeDriver(driver)
            driver.state = finalState
            driver.exception = exception
            // 把Driver所在的worker信息中移除该Driver信息
            driver.worker.foreach(w => w.removeDriver(driver))
            // 由于释放了资源，所以会重新进行调度
            schedule()
          case None =>
            logWarning(s"Asked to remove unknown driver: $driverId")
        }
      }
    ```

    

## Worker启动Executor流程

​	1. 由于worker启动Executor的流程跟Driver很像，所以不再详细介绍。
