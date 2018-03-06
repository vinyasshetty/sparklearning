* SparkSession is a companion object which has the builder method which returns a Builder Object,.
* Builder has fields : options which is a mutable HashMap\[String,String\] ,  extension:SparkExtensions object , userSuppliedContext:Option\[SparkContext\] which set to None
* SparkExtensions =&gt; This seems to be the the place for all the Rules Builders for different Plans.Will come back to this.
* Builder class also has methods as appName,master,config\(k,v\).
* So Fluent Interface is used to create a SparkSession object ie We have builder method inside SparkSession Companion object ,then when we call a builder method a Builder object is returned then we can use this to set various values using the setter method which sets the value\(using a HashMap ,options\) and returns the builder object ,finally when we call a getOrCreate method in builder object ,this will create a SparkSession object using the options HashMap.
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

