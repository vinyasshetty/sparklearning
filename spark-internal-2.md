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

Firstly care is taken to make sure only one SparkContext is active per JVM/application by default.If we require multiple context ,we set the \(set spark.driver.allowMultipleContexts = true\),have to come back to this.Once the SparkContext is created ,its set to :

```
 private val activeContext: AtomicReference[SparkContext] =
    new AtomicReference[SparkContext](null) .This is member of SparkContext companion object.
```

So basically a SparkContext object is created and this SparkContext object has lot of information in it in the form of several private fields.Below are the initial defaults

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

      try {
        _conf = config.clone()
        _conf.validateSettings()

        if (!_conf.contains("spark.master")) {
          throw new SparkException("A master URL must be set in your configuration")
        }
        if (!_conf.contains("spark.app.name")) {
          throw new SparkException("An application name must be set in your configuration")
        }

SparkContext object creation always expects to have a SparkConf to be sent. Now when SparkContext Object is created at that within the class all these parameters are set it to "java.util.UUID.randomUUID\(\).toString"

\*\*\*

In SparkContext ,there is check to make sure SparkConf always contains spark.app.name and spark.app.master. Now as a user if i dont give spark.app.name\(ie appName method on builder\) then in SparkSession we have a code to set  it to java.util.UUID.randomUUID\(\).toString.

Also spark.app.master ,in SparkSubmitArguments class ,is master is not set then it gets defaulted to "local\[\*\]"

\*\*

* SparkSession needs a SparkContext , Option\[SharedState\], Option\[SessionState\] , SparkSessionExtensions.

* SparkContext object creation initializes several things :

  private var \_conf: SparkConf = \_

  private var \_eventLogDir: Option\[URI\] = None

  private var \_eventLogCodec: Option\[String\] = None

  private var \_listenerBus: LiveListenerBus = \_

  private var \_env: SparkEnv = \_

  private var \_statusTracker: SparkStatusTracker = \_

  private var \_progressBar: Option\[ConsoleProgressBar\] = None

  private var \_ui: Option\[SparkUI\] = None

  private var \_hadoopConfiguration: Configuration = \_

  private var \_executorMemory: Int = \_

  private var \_schedulerBackend: SchedulerBackend = \_

  private var \_taskScheduler: TaskScheduler = \_

  private var \_heartbeatReceiver: RpcEndpointRef = \_

  @volatile private var \_dagScheduler: DAGScheduler = \_

  private var \_applicationId: String = \_

  private var \_applicationAttemptId: Option\[String\] = None

  private var \_eventLogger: Option\[EventLoggingListener\] = None

  private var \_executorAllocationManager: Option\[ExecutorAllocationManager\] = None

  private var \_cleaner: Option\[ContextCleaner\] = None

  private var \_listenerBusStarted: Boolean = false

  private var \_jars: Seq\[String\] = \_

  private var \_files: Seq\[String\] = \_

  private var \_shutdownHookRef: AnyRef = \_

  private var \_statusStore: AppStatusStore = \_

* SparkContext always requires a SparkConf ,if not given then it creates its own SparkConf .

* _private\[spark\] def conf: SparkConf = \_conf_

  _/\*\*_

  _**\* Return a copy of this SparkContext's configuration. The configuration ''cannot'' be**_

  _**\* changed at runtime.**_

  _**\*/**_

  _**def getConf: SparkConf = conf.clone\(\)**_

* `def master: String = _conf.get("spark.master")`

  `def deployMode: String = _conf.getOption("spark.submit.deployMode").getOrElse("client")`

  `def appName: String = _conf.get("spark.app.name")`

`private[spark] def isEventLogEnabled: Boolean = _conf.getBoolean("spark.eventLog.enabled", false)`

* 


