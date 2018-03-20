## Here Starts our User Class Main Method Execution Order:

I am trying to follow the default path ,so will exclude some code.Now in the ApplicationMaster a Thread is created which launches the main method of the User code and the application master waits for this thread to complete using "join".

SparkSession is a companion object which has the builder method which returns a Builder Object,.

```
So we start our main method with :
val spark = SparkSession.builder // returns a Builder object
            .appName("")  //calls config("spark.app.name",value)
            .master("")   //calls config("spark.master",value)
            .config(key,value)
            .getOrCreate()


    class Builder extends Logging {
    private[this] val options = new scala.collection.mutable.HashMap[String, String]
    private[this] val extensions = new SparkSessionExtensions
    private[this] var userSuppliedContext: Option[SparkContext] = None
    private[spark] def sparkContext(sparkContext: SparkContext): Builder = synchronized {
      userSuppliedContext = Option(sparkContext)
      this
    }
```

* Builder has fields : options which is a mutable HashMap\[String,String\] ,  extension:SparkExtensions object , userSuppliedContext:Option\[SparkContext\] which set to None.

* SparkExtensions =&gt; This seems to be the the place for all the Rules Builders for different Plans.Will come back to this.

* Builder class also has methods as appName,master,config\(k,v\).

  * So **Fluent Interface **is used to create a SparkSession object ie We have builder method inside SparkSession Companion object ,then when we call a builder method where a Builder object is returned then we can use this to set various values using the setter method which sets the value\(using a HashMap ,options\) and returns the builder object ,finally when we call a getOrCreate method in builder object .This will check for active SparkSession\(InheritableLocalThread\) or global \(Atomic\) ,first time when launching these will ne null ,hence a SparkSession object is created ,as part if it SparkContext and a SparkConf\(using the config we set in builder\) is created.Then a map field in SparkSession called initialSessionOptions is set using the builder option map.Also the global AtomicReference is set to the new SparkSession we created.

* ** &lt;Question&gt;Now wonder why we have global and active SparkSession and not just one SparkSession is created directly?May be i will know this once i get to the point as to how restart is happening?? **

* ```
   /** The active SparkSession for the current thread. */
  private val activeThreadSession = new InheritableThreadLocal[SparkSession]

  /** Reference to the root SparkSession. */
  private val defaultSession = new AtomicReference[SparkSession]

  in getOrCreate we first check if we have active or default SparkSession available ,if not then :
  ```

            // No active nor global default session. Create a new one.
            val sparkContext = userSuppliedContext.getOrElse {
              val sparkConf = new SparkConf()
              options.foreach { case (k, v) => sparkConf.set(k, v) } // This is created from whatever config,appname,master you have used in builder.

              // set a random app name if not given.
              if (!sparkConf.contains("spark.app.name")) {
                sparkConf.setAppName(java.util.UUID.randomUUID().toString)
              }

              SparkContext.getOrCreate(sparkConf)
              // Do not update `SparkConf` for existing `SparkContext`, as it's shared by all sessions.
            }
            session = new SparkSession(sparkContext, None, None, extensions)
            options.foreach { case (k, v) => session.initialSessionOptions.put(k, v) } //initialSessionOptions is map field
            defaultSession.set(session)  // Now global atomic reference is set.Have to come back to this.

```
        /*
        class SparkSession private(
    @transient val sparkContext: SparkContext,
    @transient private val existingSharedState: Option[SharedState],
    @transient private val parentSessionState: Option[SessionState],
    @transient private[sql] val extensions: SparkSessionExtensions){
    @transient
  private[sql] val initialSessionOptions = new scala.collection.mutable.HashMap[String, String]


    */
```

* Before moving further ahead ,i will follow the SparkContext.getOrCreate\(sparkconf\) path.

## SparkConf

A SparkConf has two important fields :

1. loadDefaults: Boolean
2. private val settings = new ConcurrentHashMap\[String, String\]\(\)

SparkConf takes a boolean as input when you create a object if you don't give anything then its set to true,basically that boolean is used to determine whether to add the Java System Properties starting with "spark." into SparkConf settings hashmap.

Settings hashmap has keys like "spark.app.name" and its corresponding value in the value part.It has a several setters and getters.All its Setters returns a SparkConf and hence we can build a chain.

## SparkContext:

SparkContext always runs on the driver and its the gateway between driver and the executors.

Firstly care is taken to make sure only one SparkContext is active per JVM/application by default.If we require multiple context ,we need to set the \(set spark.driver.allowMultipleContexts = true\),have to come back to this.Once the SparkContext is created ,its set to :

```
 private val activeContext: AtomicReference[SparkContext] =
    new AtomicReference[SparkContext](null) .This is member of SparkContext companion object.
```

So basically a SparkContext object is created and this SparkContext object has lot of information in it in the form of several private fields.**SparkContext object creation always expects to have a SparkConf to be sent.If the SparkConf has values set then those parameters take the highest precedence. Then all the Java System Properties are set into conf\(Have question on this :**

[https://stackoverflow.com/questions/49285615/spark-config-internal\](https://stackoverflow.com/questions/49285615/spark-config-internal%29\) .

Below are the initial defaults of the private fields in SparkContext.

    /**
     * Main entry point for Spark functionality. A SparkContext represents the connection to a Spark
     * cluster, and can be used to create RDDs, accumulators and broadcast variables on that cluster.
     *
     * Only one SparkContext may be active per JVM.  You must `stop()` the active SparkContext before
     * creating a new one.  This limitation may eventually be removed; see SPARK-2243 for more details.
     *
     * @param config a Spark Config object describing the application configuration. Any settings in
     *   this config overrides the default configs as well as system properties.
     */
    class SparkContext(config: SparkConf) extends Logging {
      private var _conf: SparkConf = _
      private var _eventLogDir: Option[URI] = None
      private var _eventLogCodec: Option[String] = None
      private var _listenerBus: LiveListenerBus = _
      private var _env: SparkEnv = _
      private var _statusTracker: SparkStatusTracker = _
      private var _progressBar: Option[ConsoleProgressBar] = None
      private var _ui: Option[SparkUI] = None
      private var _hadoopConfiguration: Configuration = _
      private var _executorMemory: Int = _
      private var _schedulerBackend: SchedulerBackend = _
      private var _taskScheduler: TaskScheduler = _
      private var _heartbeatReceiver: RpcEndpointRef = _
      @volatile private var _dagScheduler: DAGScheduler = _
      private var _applicationId: String = _
      private var _applicationAttemptId: Option[String] = None
      private var _eventLogger: Option[EventLoggingListener] = None
      private var _executorAllocationManager: Option[ExecutorAllocationManager] = None
      private var _cleaner: Option[ContextCleaner] = None
      private var _listenerBusStarted: Boolean = false
      private var _jars: Seq[String] = _
      private var _files: Seq[String] = _
      private var _shutdownHookRef: AnyRef = _
      private var _statusStore: AppStatusStore = _


      /**
       * Default min number of partitions for Hadoop RDDs when not given by user
       * Notice that we use math.min so the "defaultMinPartitions" cannot be higher than 2.
       * The reasons for this are discussed in https://github.com/mesos/spark/pull/718
       */
      def defaultMinPartitions: Int = math.min(defaultParallelism, 2)

Now SparkContext starts setting values for above parameters:

As you see a clone of SparkConf is given to SparkContext ,so basically once u set the SparkConf and pass it to SparkContext,trying to get the conf back from sparkcontext and changing its values will NOT work. SparkConf is set only once.

```
try {
_conf = config.clone()  
_conf.validateSettings()
```

In SparkContext ,there is check to make sure SparkConf always contains spark.app.name and spark.app.master. Now as a user if i dont give spark.app.name\(ie appName method on builder\) then in SparkSession we have a code to set  it to java.util.UUID.randomUUID\(\).toString.Also spark.app.master ,in SparkSubmitArguments class ,is master is not set then it gets defaulted to "local\[\*\]"

```
if (!_conf.contains("spark.master")) {
throw new SparkException("A master URL must be set in your configuration")
}
if (!_conf.contains("spark.app.name")) {
throw new SparkException("An application name must be set in your configuration")
}
logInfo(s"Submitted application: $appName")
```

Below spark.yarn.app.id is set to applicationId in ApplicationMaster code.

```
 // System property spark.yarn.app.id must be set if user code ran by AM on a YARN cluster
    if (master == "yarn" && deployMode == "cluster" && !_conf.contains("spark.yarn.app.id")) {
      throw new SparkException("Detected yarn cluster mode, but isn't running on a cluster. " +
        "Deployment to YARN is not supported directly by SparkContext. Please use spark-submit.")
    }
```

By setting the spark.logConf to true ,we can see all\(default included\) the conf values in our App code

```
 if (_conf.getBoolean("spark.logConf", false)) {
      logInfo("Spark configuration:\n" + _conf.toDebugString)
    }
```

Since SparkContext is created in the driver ,here the executor id is called as "driver"

```
val DRIVER_IDENTIFIER = "driver"
  _conf.set("spark.executor.id", SparkContext.DRIVER_IDENTIFIER)
```

Basically all the events of your application is logged ,use a hdfs directory.Create the directory first .Inside that directory spark will create one more directory with name as applicationid and log info.U can turn on compression also.It will use whatever is set in spark.io.compression.codec

```
  private[spark] def isEventLogEnabled: Boolean = _conf.getBoolean("spark.eventLog.enabled", false)
   _eventLogDir =
      if (isEventLogEnabled) {
        val unresolvedDir = conf.get("spark.eventLog.dir", EventLoggingListener.DEFAULT_LOG_DIR)
          .stripSuffix("/")
        Some(Utils.resolveURI(unresolvedDir))
      } else {
        None
      }

    _eventLogCodec = {
      val compress = _conf.getBoolean("spark.eventLog.compress", false)
      if (compress && isEventLogEnabled) {
        Some(CompressionCodec.getCodecName(_conf)).map(CompressionCodec.getShortName)
      } else {
        None
      }
    }
```

We can set log level on your app using SparkContext's  below method:

```
 /** Control our logLevel. This overrides any user-defined log settings.
   * @param logLevel The desired log level as a string.
   * Valid log levels include: ALL, DEBUG, ERROR, FATAL, INFO, OFF, TRACE, WARN
   */
  def setLogLevel(logLevel: String)
```

ListenerBus Creation\(**Will Come back to this\)** :

```
 _listenerBus = new LiveListenerBus(_conf)
```

TaskScheduler Creatiion \(SInce my knowledge on yarn internals is limited,i may have to look at how the local\[\*\] mode is implemented  :

```
val (sched, ts) = SparkContext.createTaskScheduler(this, master, deployMode)
    _schedulerBackend = sched
    _taskScheduler = ts


 private def createTaskScheduler(
      sc: SparkContext,
      master: String,
      deployMode: String): (SchedulerBackend, TaskScheduler) = {
    import SparkMasterRegex._
    // When running locally, don't try to re-execute tasks on failure.
    val MAX_LOCAL_TASK_FAILURES = 1

    master match {
    case LOCAL_N_REGEX(threads) =>
        def localCpuCount: Int = Runtime.getRuntime.availableProcessors() //This will give the number of cpu/cores in ur machine
        // local[*] estimates the number of cores on the machine; local[N] uses exactly N threads.
        val threadCount = if (threads == "*") localCpuCount else threads.toInt
        if (threadCount <= 0) {
          throw new SparkException(s"Asked to run locally with $threadCount threads")
        }
        val scheduler = new TaskSchedulerImpl(sc, MAX_LOCAL_TASK_FAILURES, isLocal = true)
        val backend = new LocalSchedulerBackend(sc.getConf, scheduler, threadCount)
        scheduler.initialize(backend)
        (backend, scheduler)
```

Here we create a TaskSchedulerImpl and LocalSchedulerBackend:

Different Clusters have different SchedulerBackend Implemented.It determines scheduling across jobs.Clients should first call initialize\(\) and start\(\), then submit task sets through the runTasks method.As you can see in above code ,for local mode LocalSchedulerBackend is created and the TaskSchedulerImpl is iniitialized with it. Launching of Tasks on Executors is handled by BackEnd Implementations.

Information from Looking at the TaskSchedulerImpl :

```
val SPECULATION_INTERVAL_MS = conf.getTimeAsMs("spark.speculation.interval", "100ms")
//Seems like if a task is running for more then 100ms then it eligible to be launched somehwere else.


 // CPUs to request per task
  val CPUS_PER_TASK = conf.getInt("spark.task.cpus", 1)

  //Default Scheduling Mode is FIFO
  private val schedulingModeConf = conf.get(SCHEDULER_MODE_PROPERTY, SchedulingMode.FIFO.toString)
```

**DAGSCHEDULER:**

1. **DagScheduler forms a dag of stages for each Job**.
2. It submits stages as TaskSet to underlying implementations of TaskSchedulerImpl.
3. TaksSet can run independently on the data which already on hdfs
4. DagScheduler creates Stages by breaking the RDD graphs whenever there is shuffle ie one first part of the stage writes map output to disk and the next stage starts with reading the data from the disk.At the end each stage will only have shuffle dependencies on other stages.
5. Within a stage multiple operations are combined and these same operations are run on different partitions of the same input data.**DagScheduler also tells on which location data ,tasks should run**.
6. If a task within a stage fails then TaskSchedulerImpl restarts it a couple of time before killing the whole stage.Failures caused due to shuffle files being lost is handled by DAGScheduler since now DAGScheduler has to resubmit that stage.
7. Two types of Stages : ShuffleMapStage and ResultStage\(final stage that executes the final action ie say count ,then the final transformed RDD is taken and and the count from each partitions taken and added and returned to driver\) 

Everything created inside SparkContext runs on driver like TaskScheduler, DAGScheduler etc

Now TaskScheduler is started and  to the livelisterner a event is posted about with SparkListenerExecutorAdded

```
taskScheduler.start() // listenerBus.post(SparkListenerExecutorAdded
```

```
setupAndStartListenerBus()
//This is will add any custom listeners to livelistenerbus queue : listenerBus.addToSharedQueue(listener)
// --conf spark.extraListeners=pl.jaceklaskowski.spark.CustomSparkListener else u can also call sc.addSparkListener(<>)

 postEnvironmentUpdate()
 postApplicationStart()

 //Posts different events to livelistener bus
```

"spark.default.parallelism" if NOT set,then in localmode  its set to totalcores in the system, but i dont believe in yarn mode ,its defaulted to anything.

## DataSet Creation :

Will start with a basic example of creation of a DataSet using spark.range

```
val ds = spark.range(0,11,,1,2)
```



