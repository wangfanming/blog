---
title: Spark on Yarn框架初始化
categories:
  - 大数据
tags:
  - spark
  - yarn
date: 2024-03-17 21:14:22
---

## 前言

每次当我们写完spark代码后，通过spark-submit命令提交执行。但是这个过程中发生了哪些事情却不甚明了。今天就来看看这个过程中发生了什么。

一切从下边这段提交命令开始：

```scala
spark-submit  --class org.apache.spark.examples.SparkPi \
    --master yarn \
    --conf spark.eventLog.dir=hdfs://dbmtimehadoop/tmp/spark2 \
    --deploy-mode cluster \
    --driver-memory 4g \
    --executor-memory 2g \
    --executor-cores 1 \
    --queue thequeue \
    $SPARK_HOME/examples/jars/spark-examples*.jar \
    10

```

<!--more-->

## 过程分析

通过命令行我们可以看出来任务是通过`spark-submit`这个脚本进行提交的。所以我们从这个脚本入口进行分析。

1. 位于spark安装包的bin目录下的spark-submit脚本：

   ```sh
   #!/usr/bin/env bash
   if [ -z "${SPARK_HOME}" ]; then
     source "$(dirname "$0")"/find-spark-home
   fi
   
   # disable randomized hash for string in Python 3.3+
   export PYTHONHASHSEED=0
   
   exec "${SPARK_HOME}"/bin/spark-class org.apache.spark.deploy.SparkSubmit "$@"
   ```

   内容很简单，就是首先查找到SPARK_HOME的目录所在，继而使用`spark-class`脚本启动SparkSubmit，并将我们所有从参数都传递给了`org.apache.spark.deploy.SparkSubmit`。

2. 位于spark安装包的bin目录下的spark-class脚本：

   ```sh
   #!/usr/bin/env bash
   
   if [ -z "${SPARK_HOME}" ]; then
     source "$(dirname "$0")"/find-spark-home
   fi
   
   . "${SPARK_HOME}"/bin/load-spark-env.sh
   
   # Find the java binary
   if [ -n "${JAVA_HOME}" ]; then
     RUNNER="${JAVA_HOME}/bin/java"
   else
     if [ "$(command -v java)" ]; then
       RUNNER="java"
     else
       echo "JAVA_HOME is not set" >&2
       exit 1
     fi
   fi
   
   # Find Spark jars.
   if [ -d "${SPARK_HOME}/jars" ]; then
     SPARK_JARS_DIR="${SPARK_HOME}/jars"
   else
     SPARK_JARS_DIR="${SPARK_HOME}/assembly/target/scala-$SPARK_SCALA_VERSION/jars"
   fi
   
   if [ ! -d "$SPARK_JARS_DIR" ] && [ -z "$SPARK_TESTING$SPARK_SQL_TESTING" ]; then
     echo "Failed to find Spark jars directory ($SPARK_JARS_DIR)." 1>&2
     echo "You need to build Spark with the target \"package\" before running this program." 1>&2
     exit 1
   else
     LAUNCH_CLASSPATH="$SPARK_JARS_DIR/*"
   fi
   
   # Add the launcher build dir to the classpath if requested.
   if [ -n "$SPARK_PREPEND_CLASSES" ]; then
     LAUNCH_CLASSPATH="${SPARK_HOME}/launcher/target/scala-$SPARK_SCALA_VERSION/classes:$LAUNCH_CLASSPATH"
   fi
   
   # For tests
   if [[ -n "$SPARK_TESTING" ]]; then
     unset YARN_CONF_DIR
     unset HADOOP_CONF_DIR
   fi
   
   # The launcher library will print arguments separated by a NULL character, to allow arguments with
   # characters that would be otherwise interpreted by the shell. Read that in a while loop, populating
   # an array that will be used to exec the final command.
   #
   # The exit code of the launcher is appended to the output, so the parent shell removes it from the
   # command array and checks the value to see if the launcher succeeded.
   build_command() {
     "$RUNNER" -Xmx128m -cp "$LAUNCH_CLASSPATH" org.apache.spark.launcher.Main "$@"
     printf "%d\0" $?
   }
   
   # Turn off posix mode since it does not allow process substitution
   set +o posix
   CMD=()
   # 在使用spark-submit启动任务的过程中，将org.apache.spark.launcher.Main的main方法的输出作为启动命令，用来启动org.apache.spark.deploy.SparkSubmit
   while IFS= read -d '' -r ARG; do
     CMD+=("$ARG")
   done < <(build_command "$@")
   
   COUNT=${#CMD[@]}
   LAST=$((COUNT - 1))
   LAUNCHER_EXIT_CODE=${CMD[$LAST]}
   
   # Certain JVM failures result in errors being printed to stdout (instead of stderr), which causes
   # the code that parses the output of the launcher to get confused. In those cases, check if the
   # exit code is an integer, and if it's not, handle it as a special error case.
   if ! [[ $LAUNCHER_EXIT_CODE =~ ^[0-9]+$ ]]; then
     echo "${CMD[@]}" | head -n-1 1>&2
     exit 1
   fi
   
   if [ $LAUNCHER_EXIT_CODE != 0 ]; then
     exit $LAUNCHER_EXIT_CODE
   fi
   
   CMD=("${CMD[@]:0:$LAST}")
   exec "${CMD[@]}"
   
   ```

   上述内容，简单来说就是首先检查一下SPARK_HOME、JAVA_HOME、LAUNCH_CLASSPATH等这些环境变量是否配置，没有配置就进行使用相关脚本进行搜索，然后调用org.apache.spark.launcher.Main的main方法，并将main的输出作为最后启动org.apache.spark.deploy.SparkSubmit的参数的一部分。

3. org.apache.spark.launcher.Main

   ```
   public static void main(String[] argsArray) throws Exception {
       checkArgument(argsArray.length > 0, "Not enough arguments: missing class name.");
   
       List<String> args = new ArrayList<>(Arrays.asList(argsArray));
       String className = args.remove(0);
   
       boolean printLaunchCommand = !isEmpty(System.getenv("SPARK_PRINT_LAUNCH_COMMAND"));
       Map<String, String> env = new HashMap<>();
       List<String> cmd;
       if (className.equals("org.apache.spark.deploy.SparkSubmit")) {
         try {
           AbstractCommandBuilder builder = new SparkSubmitCommandBuilder(args);
           // 根据参数配置构建spark-submit命令
           cmd = buildCommand(builder, env, printLaunchCommand);
         ...
       if (isWindows()) {
         System.out.println(prepareWindowsCommand(cmd, env));
       } else {
         // In bash, use NULL as the arg separator since it cannot be used in an argument.
         List<String> bashCmd = prepareBashCommand(cmd, env);
         // 打印spark-submit命令,将输出作为参数传递给spark-class脚本
         for (String c : bashCmd) {
           System.out.print(c);
           System.out.print('\0');
         }
       }
     }
   ```

4. 然后org.apache.spark.deploy.SparkSubmit的main方法启动

   

   ```scala
    override def main(args: Array[String]): Unit = {
       val submit = new SparkSubmit() {
         ...
       }
   
       submit.doSubmit(args)
     }
   ```

   ```scala
   def doSubmit(args: Array[String]): Unit = {
       ...
       appArgs.action match {
         // 在SparkSubmitArguments初始化时的默认值
         case SparkSubmitAction.SUBMIT => submit(appArgs, uninitLog)
         case SparkSubmitAction.KILL => kill(appArgs)
         case SparkSubmitAction.REQUEST_STATUS => requestStatus(appArgs)
         case SparkSubmitAction.PRINT_VERSION => printVersion()
       }
     }
   ```

   ```scala
   private def submit(args: SparkSubmitArguments, uninitLog: Boolean): Unit = {
   
       def doRunMain(): Unit = {
         if (args.proxyUser != null) {
           ...
         } else {
           // 按照启动参数，启动Saprk框架
           runMain(args, uninitLog)
         }
       }
   
      
       // standalone模式下的启动过程
       if (args.isStandaloneCluster && args.useRest) {
         try {
           logInfo("Running Spark using the REST application submission protocol.")
           doRunMain()
         } catch {
           // Fail over to use the legacy submission gateway
           case e: SubmitRestConnectionException =>
             logWarning(s"Master endpoint ${args.master} was not a REST server. " +
               "Falling back to legacy submission gateway instead.")
             args.useRest = false
             submit(args, false)
         }
       // In all other modes, just run the main class as prepared
       } else {
         // 启动用户主类
         doRunMain()
       }
     }
   ```

   ```scala
    private def runMain(args: SparkSubmitArguments, uninitLog: Boolean): Unit = {
       // 解析参数，下载资源文件，设置环境变量
       val (childArgs, childClasspath, sparkConf, childMainClass) = prepareSubmitEnvironment(args)
       ...
       val loader =
         if (sparkConf.get(DRIVER_USER_CLASS_PATH_FIRST)) {
           new ChildFirstURLClassLoader(new Array[URL](0),
             Thread.currentThread.getContextClassLoader)
         } else {
           new MutableURLClassLoader(new Array[URL](0),
             Thread.currentThread.getContextClassLoader)
         }
       Thread.currentThread.setContextClassLoader(loader)
   
       for (jar <- childClasspath) {
         addJarToClasspath(jar, loader)
       }
   
       var mainClass: Class[_] = null
   
       try {
         // yarn集群模式下，mainclass:org.apache.spark.deploy.yarn.YarnClusterApplication
         mainClass = Utils.classForName(childMainClass)
       } catch {
           ...
       }
   
       val app: SparkApplication = if (classOf[SparkApplication].isAssignableFrom(mainClass)) {
         mainClass.newInstance().asInstanceOf[SparkApplication]
       } else {
         // SPARK-4170
         if (classOf[scala.App].isAssignableFrom(mainClass)) {
           logWarning("Subclasses of scala.App may not work correctly. Use a main() method instead.")
         }
         new JavaMainApplication(mainClass)
       }
   ...
   
       try {
         // yarn-cluster模式下，启动org.apache.spark.deploy.yarn.YarnClusterApplication#start
         app.start(childArgs.toArray, sparkConf)
       } catch {
         case t: Throwable =>
           throw findCause(t)
       }
     }
   ```

   - YarnClusterApplication是位于源码目录resource-managers/yarn/src/main/scala/org/apache/spark/deploy/yarn/Client.scala的内部类，在程序运行时会被加载至classpath。

   ```scala
   private[spark] class YarnClusterApplication extends SparkApplication {
   
     override def start(args: Array[String], conf: SparkConf): Unit = {
       // SparkSubmit would use yarn cache to distribute files & jars in yarn mode,
       // so remove them from sparkConf here for yarn mode.
       conf.remove("spark.jars")
       conf.remove("spark.files")
   	// 初始化Client对象，解析AM、Executor的参数,然后提交应用
       new Client(new ClientArguments(args), conf).run()
     }
   
   }	
   ```

   ```scala
   def run(): Unit = {
       // 向Yarn提交应用,启动AM,在AM启动后，由其向NodeManager申请容器启动Executor
       this.appId = submitApplication()
       ...
     }
   ```

   ```scala
   def submitApplication(): ApplicationId = {
       var appId: ApplicationId = null
       try {
         launcherBackend.connect()
         yarnClient.init(hadoopConf)
         yarnClient.start()
   
         logInfo("Requesting a new application from cluster with %d NodeManagers"
           .format(yarnClient.getYarnClusterMetrics.getNumNodeManagers))
   
         // Get a new application from our RM
         val newApp = yarnClient.createApplication()
         val newAppResponse = newApp.getNewApplicationResponse()
         appId = newAppResponse.getApplicationId()
   
         new CallerContext("CLIENT", sparkConf.get(APP_CALLER_CONTEXT),
           Option(appId.toString)).setCurrentContext()
   
         // Verify whether the cluster has enough resources for our AM
         // 检查RM资源是否可以满足AM的需求
         verifyClusterResources(newAppResponse)
   
         // Set up the appropriate contexts to launch our AM
         // 配置AM的环境变量，上传资源文件到获得方式，拼凑AM的启动命令
         val containerContext = createContainerLaunchContext(newAppResponse)
         val appContext = createApplicationSubmissionContext(newApp, containerContext)
   
         // Finally, submit and monitor the application
         logInfo(s"Submitting application $appId to ResourceManager")
         // 启动AM:ApplicationMaster,AM类的main方法会被调用,继而开始进行Executor的申请和启动,在cluster模式下，用户的Driver线程会在AM的main方法中启动
         yarnClient.submitApplication(appContext)
         launcherBackend.setAppId(appId.toString)
         reportLauncherState(SparkAppHandle.State.SUBMITTED)
   
         appId
       } catch {
         case e: Throwable =>
           if (appId != null) {
             cleanupStagingDir(appId)
           }
           throw e
       }
     }
   ```

   - 到这里，ApplicationMaster类的main方法开始启动。在Yarn-cluster模式下，接下来会开始启动Executor和用户编写的应用程序。

5. 启动Executor和用户编写的应用程序

   org.apache.spark.deploy.yarn.ApplicationMaster#main：

   ```scala
   def main(args: Array[String]): Unit = {
       SignalUtils.registerLogger(log)
       val amArgs = new ApplicationMasterArguments(args)
       master = new ApplicationMaster(amArgs)
       System.exit(master.run())
     }
   
   final def run(): Int = {
       doAsUser {
         runImpl()
       }
       exitCode
     }
   
   private def runImpl(): Unit = {
       try {
         val appAttemptId = client.getAttemptId()
         ...
         if (isClusterMode) {
           // 启动用户类线程
           runDriver()
         } else {
           runExecutorLauncher()
         }
       } catch {
         ...
     }
   ```

   ```scala
   private def runDriver(): Unit = {
       addAmIpFilter(None)
       // 在用户类线程中启动SparkContext，启动用户编写的Spark应用
       userClassThread = startUserApplication()
   
       // This a bit hacky, but we need to wait until the spark.driver.port property has
       // been set by the Thread executing the user class.
       logInfo("Waiting for spark context initialization...")
       val totalWaitTime = sparkConf.get(AM_MAX_WAIT_TIME)
       try {
         // 等待SparkContext初始化，由sparkContextPromise来保证
         val sc = ThreadUtils.awaitResult(sparkContextPromise.future,
           Duration(totalWaitTime, TimeUnit.MILLISECONDS))
         if (sc != null) {
           rpcEnv = sc.env.rpcEnv
   
           val userConf = sc.getConf
           val host = userConf.get("spark.driver.host")
           val port = userConf.get("spark.driver.port").toInt
           registerAM(host, port, userConf, sc.ui.map(_.webUrl))
   
           val driverRef = rpcEnv.setupEndpointRef(
             RpcAddress(host, port),
             YarnSchedulerBackend.ENDPOINT_NAME)
           // 创建资源分配器，启动Executor
           createAllocator(driverRef, userConf)
         } else {
           // Sanity check; should never happen in normal operation, since sc should only be null
           // if the user app did not create a SparkContext.
           throw new IllegalStateException("User did not initialize spark context!")
         }
         // Executor已经启动，恢复用户类线程
         resumeDriver()
         // 等待用户类线程结束
         userClassThread.join()
       } catch {
         case e: SparkException if e.getCause().isInstanceOf[TimeoutException] =>
           logError(
             s"SparkContext did not initialize after waiting for $totalWaitTime ms. " +
              "Please check earlier log output for errors. Failing the application.")
           finish(FinalApplicationStatus.FAILED,
             ApplicationMaster.EXIT_SC_NOT_INITED,
             "Timed out waiting for SparkContext.")
       } finally {
         resumeDriver()
       }
     }
   ```

   ```scala
     private def createAllocator(driverRef: RpcEndpointRef, _sparkConf: SparkConf): Unit = {
       ...
       // YarnRMClient
       allocator = client.createAllocator(
         yarnConf,
         _sparkConf,
         driverUrl,
         driverRef,
         securityMgr,
         localResources)
   
       credentialRenewer.foreach(_.setDriverRef(driverRef))
   
       // Initialize the AM endpoint *after* the allocator has been initialized. This ensures
       // that when the driver sends an initial executor request (e.g. after an AM restart),
       // the allocator is ready to service requests.
       rpcEnv.setupEndpoint("YarnAM", new AMEndpoint(rpcEnv, driverRef))
   
       // 在分配到的容器内启动Executor进程
       allocator.allocateResources()
       val ms = MetricsSystem.createMetricsSystem("applicationMaster", sparkConf, securityMgr)
       val prefix = _sparkConf.get(YARN_METRICS_NAMESPACE).getOrElse(appId)
       ms.registerSource(new ApplicationMasterSource(prefix, allocator))
       // do not register static sources in this case as per SPARK-25277
       ms.start(false)
       metricsSystem = Some(ms)
       reporterThread = launchReporterThread()
     }
   ```

   ```scala
     def allocateResources(): Unit = synchronized {
       ...
   		// 处理已分配到的容器
         handleAllocatedContainers(allocatedContainers.asScala)
       }
   	...
   ```

   ```scala
     def handleAllocatedContainers(allocatedContainers: Seq[Container]): Unit = {
       ...
       // 在容器中启动executor
       runAllocatedContainers(containersToUse)
   
       ...
     }
   ```

   ```scala
     private def runAllocatedContainers(containersToUse: ArrayBuffer[Container]): Unit = {
       for (container <- containersToUse) {
   ...
   
         if (runningExecutors.size() < targetNumExecutors) {
           numExecutorsStarting.incrementAndGet()
           if (launchContainers) {
             // 使用线程池启动executor
             launcherPool.execute(new Runnable {
               override def run(): Unit = {
                 try {
                   new ExecutorRunnable(
                     Some(container),
                     conf,
                     sparkConf,
                     driverUrl,
                     executorId,
                     executorHostname,
                     executorMemory,
                     executorCores,
                     appAttemptId.getApplicationId.toString,
                     securityMgr,
                     localResources
                   ).run()
                   updateInternalState()
                 } catch {
             ...
       }
     }
   ```

6. 接下来开始Executor的启动过程

   org.apache.spark.deploy.yarn.ExecutorRunnable#run

   ```scala
     def run(): Unit = {
       logDebug("Starting Executor Container")
       nmClient = NMClient.createNMClient()
       nmClient.init(conf)
       nmClient.start()
       startContainer()
     }
   ```

   ```scala
     def startContainer(): java.util.Map[String, ByteBuffer] = {
       val ctx = Records.newRecord(classOf[ContainerLaunchContext])
         .asInstanceOf[ContainerLaunchContext]
       val env = prepareEnvironment().asJava
   
       ctx.setLocalResources(localResources.asJava)
       ctx.setEnvironment(env)
   
       val credentials = UserGroupInformation.getCurrentUser().getCredentials()
       val dob = new DataOutputBuffer()
       credentials.writeTokenStorageToStream(dob)
       ctx.setTokens(ByteBuffer.wrap(dob.getData()))
       // 启动Executor的主进程CoarseGrainedExecutorBackend
       val commands = prepareCommand()
   
       ctx.setCommands(commands.asJava)
       ctx.setApplicationACLs(
         YarnSparkHadoopUtil.getApplicationAclsForYarn(securityMgr).asJava)
   
       // If external shuffle service is enabled, register with the Yarn shuffle service already
       // started on the NodeManager and, if authentication is enabled, provide it with our secret
       // key for fetching shuffle files later
       if (sparkConf.get(SHUFFLE_SERVICE_ENABLED)) {
         val secretString = securityMgr.getSecretKey()
         val secretBytes =
           if (secretString != null) {
             // This conversion must match how the YarnShuffleService decodes our secret
             JavaUtils.stringToBytes(secretString)
           } else {
             // Authentication is not enabled, so just provide dummy metadata
             ByteBuffer.allocate(0)
           }
         ctx.setServiceData(Collections.singletonMap("spark_shuffle", secretBytes))
       }
   
       // Send the start request to the ContainerManager
       try {
         // 通过NodeManager客户端，启动具体的容器进程
         nmClient.startContainer(container.get, ctx)
       } catch {
         case ex: Exception =>
           throw new SparkException(s"Exception while starting container ${container.get.getId}" +
             s" on host $hostname", ex)
       }
     }
   ```

   ```scala
     private def prepareCommand(): List[String] = {
       // Extra options for the JVM
       val javaOpts = ListBuffer[String]()
   
       // Set the JVM memory
       val executorMemoryString = executorMemory + "m"
       javaOpts += "-Xmx" + executorMemoryString
   
       // Set extra Java options for the executor, if defined
       sparkConf.get(EXECUTOR_JAVA_OPTIONS).foreach { opts =>
         val subsOpt = Utils.substituteAppNExecIds(opts, appId, executorId)
         javaOpts ++= Utils.splitCommandString(subsOpt).map(YarnSparkHadoopUtil.escapeForShell)
       }
   
       // Set the library path through a command prefix to append to the existing value of the
       // env variable.
       val prefixEnv = sparkConf.get(EXECUTOR_LIBRARY_PATH).map { libPath =>
         Client.createLibraryPathPrefix(libPath, sparkConf)
       }
   
       javaOpts += "-Djava.io.tmpdir=" +
         new Path(Environment.PWD.$$(), YarnConfiguration.DEFAULT_CONTAINER_TEMP_DIR)
   
       // Certain configs need to be passed here because they are needed before the Executor
       // registers with the Scheduler and transfers the spark configs. Since the Executor backend
       // uses RPC to connect to the scheduler, the RPC settings are needed as well as the
       // authentication settings.
       sparkConf.getAll
         .filter { case (k, v) => SparkConf.isExecutorStartupConf(k) }
         .foreach { case (k, v) => javaOpts += YarnSparkHadoopUtil.escapeForShell(s"-D$k=$v") }
   
       // Commenting it out for now - so that people can refer to the properties if required. Remove
       // it once cpuset version is pushed out.
       // The context is, default gc for server class machines end up using all cores to do gc - hence
       // if there are multiple containers in same node, spark gc effects all other containers
       // performance (which can also be other spark containers)
       // Instead of using this, rely on cpusets by YARN to enforce spark behaves 'properly' in
       // multi-tenant environments. Not sure how default java gc behaves if it is limited to subset
       // of cores on a node.
       /*
           else {
             // If no java_opts specified, default to using -XX:+CMSIncrementalMode
             // It might be possible that other modes/config is being done in
             // spark.executor.extraJavaOptions, so we don't want to mess with it.
             // In our expts, using (default) throughput collector has severe perf ramifications in
             // multi-tenant machines
             // The options are based on
             // http://www.oracle.com/technetwork/java/gc-tuning-5-138395.html#0.0.0.%20When%20to%20Use
             // %20the%20Concurrent%20Low%20Pause%20Collector|outline
             javaOpts += "-XX:+UseConcMarkSweepGC"
             javaOpts += "-XX:+CMSIncrementalMode"
             javaOpts += "-XX:+CMSIncrementalPacing"
             javaOpts += "-XX:CMSIncrementalDutyCycleMin=0"
             javaOpts += "-XX:CMSIncrementalDutyCycle=10"
           }
       */
   
       // For log4j configuration to reference
       javaOpts += ("-Dspark.yarn.app.container.log.dir=" + ApplicationConstants.LOG_DIR_EXPANSION_VAR)
   
       val userClassPath = Client.getUserClasspath(sparkConf).flatMap { uri =>
         val absPath =
           if (new File(uri.getPath()).isAbsolute()) {
             Client.getClusterPath(sparkConf, uri.getPath())
           } else {
             Client.buildPath(Environment.PWD.$(), uri.getPath())
           }
         Seq("--user-class-path", "file:" + absPath)
       }.toSeq
   
       YarnSparkHadoopUtil.addOutOfMemoryErrorArgument(javaOpts)
       val commands = prefixEnv ++
         Seq(Environment.JAVA_HOME.$$() + "/bin/java", "-server") ++
         javaOpts ++
       // Executor进程的主类：CoarseGrainedExecutorBackend
         Seq("org.apache.spark.executor.CoarseGrainedExecutorBackend",
           "--driver-url", masterAddress,
           "--executor-id", executorId,
           "--hostname", hostname,
           "--cores", executorCores.toString,
           "--app-id", appId) ++
         userClassPath ++
         Seq(
           s"1>${ApplicationConstants.LOG_DIR_EXPANSION_VAR}/stdout",
           s"2>${ApplicationConstants.LOG_DIR_EXPANSION_VAR}/stderr")
   
       // TODO: it would be nicer to just make sure there are no null commands here
       commands.map(s => if (s == null) "null" else s).toList
     }
   ```

7. 接下来进行CoarseGrainedExecutorBackend的初始化过程，这里就涉及到了消息系统的初始化过程。

   ```scala
   private[spark] object CoarseGrainedExecutorBackend extends Logging {
     def main(args: Array[String]) {
       ...
       run(driverUrl, executorId, hostname, cores, appId, workerUrl, userClassPath)
       System.exit(0)
     }
     
   private def run(
         driverUrl: String,
         executorId: String,
         hostname: String,
         cores: Int,
         appId: String,
         workerUrl: Option[String],
         userClassPath: Seq[URL]) {
   
      ...
      // 这里初始化了一个基于Netty的的消息系统，并且在初始化时会创建一个线程池，用于处理消息，且发送类一个OnStart事件用来启动消息处理
         val env = SparkEnv.createExecutorEnv(
           driverConf, executorId, hostname, cores, cfg.ioEncryptionKey, isLocal = false)
   
         env.rpcEnv.setupEndpoint("Executor", new CoarseGrainedExecutorBackend(
           env.rpcEnv, driverUrl, executorId, hostname, cores, userClassPath, env))
         workerUrl.foreach { url =>
           env.rpcEnv.setupEndpoint("WorkerWatcher", new WorkerWatcher(env.rpcEnv, url))
         }
       // 启动消息处理线程
         env.rpcEnv.awaitTermination()
       }
     }
   
   // 以下为消息处理线程相关信息，它们主要来自于org.apache.spark.rpc.netty.Dispatcher
   override def awaitTermination(): Unit = {
       dispatcher.awaitTermination()
     }
   def awaitTermination(): Unit = {
       threadpool.awaitTermination(Long.MaxValue, TimeUnit.MILLISECONDS)
     }
   private val threadpool: ThreadPoolExecutor = {
       val availableCores =
         if (numUsableCores > 0) numUsableCores else Runtime.getRuntime.availableProcessors()
       val numThreads = nettyEnv.conf.getInt("spark.rpc.netty.dispatcher.numThreads",
         math.max(2, availableCores))
       val pool = ThreadUtils.newDaemonFixedThreadPool(numThreads, "dispatcher-event-loop")
       for (i <- 0 until numThreads) {
         pool.execute(new MessageLoop)
       }
       pool
     }
     private class MessageLoop extends Runnable {
       override def run(): Unit = {
         try {
           while (true) {
             try {
               val data = receivers.take()
               if (data == PoisonPill) {
                 // Put PoisonPill back so that other MessageLoops can see it.
                 receivers.offer(PoisonPill)
                 return
               }
                 // 开始处理消息
               data.inbox.process(Dispatcher.this)
             } catch {
               case NonFatal(e) => logError(e.getMessage, e)
             }
           }
         } catch {
           case _: InterruptedException => // exit
           case t: Throwable =>
             try {
               // Re-submit a MessageLoop so that Dispatcher will still work if
               // UncaughtExceptionHandler decides to not kill JVM.
               threadpool.execute(new MessageLoop)
             } finally {
               throw t
             }
         }
       }
     }
   ```

8. 创建一个基于Netty的消息系统

   org.apache.spark.SparkEnv#createExecutorEnv:

   ```scala
     private[spark] def createExecutorEnv(
         conf: SparkConf,
         executorId: String,
         hostname: String,
         numCores: Int,
         ioEncryptionKey: Option[Array[Byte]],
         isLocal: Boolean): SparkEnv = {
             // 初始化Executor的serializer,
       //      closureSerializer,
       //      serializerManager,
       //      mapOutputTracker,
       //      shuffleManager,
       //      broadcastManager,
       //      blockManager,
       //      securityManager,
       //      metricsSystem,
       //      memoryManager,
       //      outputCommitCoordinator组建，启动消息系统
       val env = create(
         conf,
         executorId,
         hostname,
         hostname,
         None,
         isLocal,
         numCores,
         ioEncryptionKey
       )
       SparkEnv.set(env)
       env
     }
   
   private def create(
         conf: SparkConf,
         executorId: String,
         bindAddress: String,
         advertiseAddress: String,
         port: Option[Int],
         isLocal: Boolean,
         numUsableCores: Int,
         ioEncryptionKey: Option[Array[Byte]],
         listenerBus: LiveListenerBus = null,
         mockOutputCommitCoordinator: Option[OutputCommitCoordinator] = None): SparkEnv = {
   
       val isDriver = executorId == SparkContext.DRIVER_IDENTIFIER
   
       // Listener bus is only used on the driver
       if (isDriver) {
         assert(listenerBus != null, "Attempted to create driver SparkEnv with null listener bus!")
       }
   
       val securityManager = new SecurityManager(conf, ioEncryptionKey)
       if (isDriver) {
         securityManager.initializeAuth()
       }
   
       ioEncryptionKey.foreach { _ =>
         if (!securityManager.isEncryptionEnabled()) {
           logWarning("I/O encryption enabled without RPC encryption: keys will be visible on the " +
             "wire.")
         }
       }
   
       val systemName = if (isDriver) driverSystemName else executorSystemName
       // 创建一个基于Netty消息系统的分布式执行环境
       val rpcEnv = RpcEnv.create(systemName, bindAddress, advertiseAddress, port.getOrElse(-1), conf,
         securityManager, numUsableCores, !isDriver)
       ...
   
       val serializer = instantiateClassFromConf[Serializer](
         "spark.serializer", "org.apache.spark.serializer.JavaSerializer")
       logDebug(s"Using serializer: ${serializer.getClass}")
   
       val serializerManager = new SerializerManager(serializer, conf, ioEncryptionKey)
   ...
   
       val broadcastManager = new BroadcastManager(isDriver, conf, securityManager)
   
       val mapOutputTracker = if (isDriver) {
         new MapOutputTrackerMaster(conf, broadcastManager, isLocal)
       } else {
         new MapOutputTrackerWorker(conf)
       }
   
       // Have to assign trackerEndpoint after initialization as MapOutputTrackerEndpoint
       // requires the MapOutputTracker itself
       mapOutputTracker.trackerEndpoint = registerOrLookupEndpoint(MapOutputTracker.ENDPOINT_NAME,
         new MapOutputTrackerMasterEndpoint(
           rpcEnv, mapOutputTracker.asInstanceOf[MapOutputTrackerMaster], conf))
   
       // Let the user specify short names for shuffle managers
       val shortShuffleMgrNames = Map(
         "sort" -> classOf[org.apache.spark.shuffle.sort.SortShuffleManager].getName,
         "tungsten-sort" -> classOf[org.apache.spark.shuffle.sort.SortShuffleManager].getName)
       val shuffleMgrName = conf.get("spark.shuffle.manager", "sort")
       val shuffleMgrClass =
         shortShuffleMgrNames.getOrElse(shuffleMgrName.toLowerCase(Locale.ROOT), shuffleMgrName)
       val shuffleManager = instantiateClass[ShuffleManager](shuffleMgrClass)
   
       val useLegacyMemoryManager = conf.getBoolean("spark.memory.useLegacyMode", false)
       val memoryManager: MemoryManager =
         if (useLegacyMemoryManager) {
           new StaticMemoryManager(conf, numUsableCores)
         } else {
           UnifiedMemoryManager(conf, numUsableCores)
         }
   
       val blockManagerPort = if (isDriver) {
         conf.get(DRIVER_BLOCK_MANAGER_PORT)
       } else {
         conf.get(BLOCK_MANAGER_PORT)
       }
   
       val blockTransferService =
         new NettyBlockTransferService(conf, securityManager, bindAddress, advertiseAddress,
           blockManagerPort, numUsableCores)
   
       val blockManagerMaster = new BlockManagerMaster(registerOrLookupEndpoint(
         BlockManagerMaster.DRIVER_ENDPOINT_NAME,
         new BlockManagerMasterEndpoint(rpcEnv, isLocal, conf, listenerBus)),
         conf, isDriver)
   
       // NB: blockManager is not valid until initialize() is called later.
       val blockManager = new BlockManager(executorId, rpcEnv, blockManagerMaster,
         serializerManager, conf, memoryManager, mapOutputTracker, shuffleManager,
         blockTransferService, securityManager, numUsableCores)
   
       val metricsSystem = if (isDriver) {
         // Don't start metrics system right now for Driver.
         // We need to wait for the task scheduler to give us an app ID.
         // Then we can start the metrics system.
         MetricsSystem.createMetricsSystem("driver", conf, securityManager)
       } else {
         // We need to set the executor ID before the MetricsSystem is created because sources and
         // sinks specified in the metrics configuration file will want to incorporate this executor's
         // ID into the metrics they report.
         conf.set("spark.executor.id", executorId)
         val ms = MetricsSystem.createMetricsSystem("executor", conf, securityManager)
         ms.start()
         ms
       }
   
       val outputCommitCoordinator = mockOutputCommitCoordinator.getOrElse {
         new OutputCommitCoordinator(conf, isDriver)
       }
       val outputCommitCoordinatorRef = registerOrLookupEndpoint("OutputCommitCoordinator",
         new OutputCommitCoordinatorEndpoint(rpcEnv, outputCommitCoordinator))
       outputCommitCoordinator.coordinatorRef = Some(outputCommitCoordinatorRef)
   
       val envInstance = new SparkEnv(
         executorId,
         rpcEnv,
         serializer,
         closureSerializer,
         serializerManager,
         mapOutputTracker,
         shuffleManager,
         broadcastManager,
         blockManager,
         securityManager,
         metricsSystem,
         memoryManager,
         outputCommitCoordinator,
         conf)
   
       // Add a reference to tmp dir created by driver, we will delete this tmp dir when stop() is
       // called, and we only need to do it for driver. Because driver may run as a service, and if we
       // don't delete this tmp dir when sc is stopped, then will create too many tmp dirs.
       if (isDriver) {
         val sparkFilesDir = Utils.createTempDir(Utils.getLocalDir(conf), "userFiles").getAbsolutePath
         envInstance.driverTmpDir = Some(sparkFilesDir)
       }
   
       envInstance
     }
   ```

9. RpcEnv初始化过程

   - org.apache.spark.rpc.RpcEnv#create:

     ```scala
       def create(
           name: String,
           bindAddress: String,
           advertiseAddress: String,
           port: Int,
           conf: SparkConf,
           securityManager: SecurityManager,
           numUsableCores: Int,
           clientMode: Boolean): RpcEnv = {
         val config = RpcEnvConfig(conf, name, bindAddress, advertiseAddress, port, securityManager,
           numUsableCores, clientMode)
         new NettyRpcEnvFactory().create(config)
       }
     ```

   - org.apache.spark.rpc.netty.NettyRpcEnvFactory#create

     ```scala
       def create(config: RpcEnvConfig): RpcEnv = {
         val sparkConf = config.conf
         // Use JavaSerializerInstance in multiple threads is safe. However, if we plan to support
         // KryoSerializer in future, we have to use ThreadLocal to store SerializerInstance
         val javaSerializerInstance =
           new JavaSerializer(sparkConf).newInstance().asInstanceOf[JavaSerializerInstance]
     
         // 创建基于Netty框架的消息系统
         val nettyEnv =
           new NettyRpcEnv(sparkConf, javaSerializerInstance, config.advertiseAddress,
             config.securityManager, config.numUsableCores)
         if (!config.clientMode) {
           val startNettyRpcEnv: Int => (NettyRpcEnv, Int) = { actualPort =>
             // 向外部系统注册RPC服务
             nettyEnv.startServer(config.bindAddress, actualPort)
             (nettyEnv, nettyEnv.address.port)
           }
           try {
             Utils.startServiceOnPort(config.port, startNettyRpcEnv, sparkConf, config.name)._1
           } catch {
             case NonFatal(e) =>
               nettyEnv.shutdown()
               throw e
           }
         }
         nettyEnv
       }
     ```

   - org.apache.spark.rpc.netty.Dispatcher#registerRpcEndpoint

     ```scala
       def registerRpcEndpoint(name: String, endpoint: RpcEndpoint): NettyRpcEndpointRef = {
         val addr = RpcEndpointAddress(nettyEnv.address, name)
         val endpointRef = new NettyRpcEndpointRef(nettyEnv.conf, addr, nettyEnv)
         synchronized {
           if (stopped) {
             throw new IllegalStateException("RpcEnv has been stopped")
           }
           if (endpoints.putIfAbsent(name, new EndpointData(name, endpoint, endpointRef)) != null) {
             throw new IllegalArgumentException(s"There is already an RpcEndpoint called $name")
           }
           val data = endpoints.get(name)
           endpointRefs.put(data.endpoint, data.ref)
           receivers.offer(data)  // for the OnStart message
         }
         endpointRef
       }
     ```

   - EndpointData初始化

     ```scala
       private class EndpointData(
           val name: String,
           val endpoint: RpcEndpoint,
           val ref: NettyRpcEndpointRef) {
         val inbox = new Inbox(ref, endpoint)
       }
     
     private[netty] class Inbox(
         val endpointRef: NettyRpcEndpointRef,
         val endpoint: RpcEndpoint)
       extends Logging {
     
       inbox =>  // Give this an alias so we can use it more clearly in closures.
     
       @GuardedBy("this")
       protected val messages = new java.util.LinkedList[InboxMessage]()
     
       /** True if the inbox (and its associated endpoint) is stopped. */
       @GuardedBy("this")
       private var stopped = false
     
       /** Allow multiple threads to process messages at the same time. */
       @GuardedBy("this")
       private var enableConcurrent = false
     
       /** The number of threads processing messages for this inbox. */
       @GuardedBy("this")
       private var numActiveThreads = 0
     
       // OnStart should be the first message to process
       inbox.synchronized {
         // 向消息队列内添加OnStart事件
         messages.add(OnStart)
       }
     
     ```

   - Executor消息系统接收到OnStart事件后向Driver注册自己的信息,

     org.apache.spark.executor.CoarseGrainedExecutorBackend#onStart:

     ```scala
       override def onStart() {
         logInfo("Connecting to driver: " + driverUrl)
         rpcEnv.asyncSetupEndpointRefByURI(driverUrl).flatMap { ref =>
           // This is a very fast action so we can use "ThreadUtils.sameThread"
           driver = Some(ref)
           ref.ask[Boolean](RegisterExecutor(executorId, self, hostname, cores, extractLogUrls))
         }(ThreadUtils.sameThread).onComplete {
           // This is a very fast action so we can use "ThreadUtils.sameThread"
           case Success(msg) =>
             // Always receive `true`. Just ignore it
           case Failure(e) =>
             exitExecutor(1, s"Cannot register with driver: $driverUrl", e, notifyDriver = false)
         }(ThreadUtils.sameThread)
       }
     ```

   - Driver接收到RegisterExecutor事件的处理，org.apache.spark.scheduler.cluster.CoarseGrainedSchedulerBackend.DriverEndpoint#receiveAndReply：

     ```scala
     override def receiveAndReply(context: RpcCallContext): PartialFunction[Any, Unit] = {
     
           case RegisterExecutor(executorId, executorRef, hostname, cores, logUrls) =>
             if (executorDataMap.contains(executorId)) {
               executorRef.send(RegisterExecutorFailed("Duplicate executor ID: " + executorId))
               context.reply(true)
             } else if (scheduler.nodeBlacklist.contains(hostname)) {
               // If the cluster manager gives us an executor on a blacklisted node (because it
               // already started allocating those resources before we informed it of our blacklist,
               // or if it ignored our blacklist), then we reject that executor immediately.
               logInfo(s"Rejecting $executorId as it has been blacklisted.")
               executorRef.send(RegisterExecutorFailed(s"Executor is blacklisted: $executorId"))
               context.reply(true)
             } else {
               // If the executor's rpc env is not listening for incoming connections, `hostPort`
               // will be null, and the client connection should be used to contact the executor.
               val executorAddress = if (executorRef.address != null) {
                   executorRef.address
                 } else {
                   context.senderAddress
                 }
               logInfo(s"Registered executor $executorRef ($executorAddress) with ID $executorId")
               addressToExecutorId(executorAddress) = executorId
               totalCoreCount.addAndGet(cores)
               totalRegisteredExecutors.addAndGet(1)
               val data = new ExecutorData(executorRef, executorAddress, hostname,
                 cores, cores, logUrls)
               // This must be synchronized because variables mutated
               // in this block are read when requesting executors
               CoarseGrainedSchedulerBackend.this.synchronized {
                 executorDataMap.put(executorId, data)
                 if (currentExecutorIdCounter < executorId.toInt) {
                   currentExecutorIdCounter = executorId.toInt
                 }
                 if (numPendingExecutors > 0) {
                   numPendingExecutors -= 1
                   logDebug(s"Decremented number of pending executors ($numPendingExecutors left)")
                 }
               }
                 // 向Executor发送RegisteredExecutor事件
               executorRef.send(RegisteredExecutor)
               // Note: some tests expect the reply to come after we put the executor in the map
               context.reply(true)
               listenerBus.post(
                 SparkListenerExecutorAdded(System.currentTimeMillis(), executorId, data))
               makeOffers()
             }
     
     ```

   - Executor接收到Driver回复的RegisteredExecutor消息后的处理：

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
     
         case RegisterExecutorFailed(message) =>
           exitExecutor(1, "Slave registration failed: " + message)
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

   至此，spark的AM、Executor都已经成功启动，等待用户程序被解析成DAG后，生成TaskSet交由Executor开始执行。

