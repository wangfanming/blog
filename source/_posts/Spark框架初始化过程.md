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

至此，AM、Executor、用户编写的应用程序都启动起来类。关于用户应用程序的初始化过程，下次再分析。
