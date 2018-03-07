* SparkSession is a companion object which has the builder method which returns a Builder Object,.
* Builder has fields : options which is a mutable HashMap\[String,String\] ,  extension:SparkExtensions object , userSuppliedContext:Option\[SparkContext\] which set to None
* SparkExtensions =&gt; This seems to be the the place for all the Rules Builders for different Plans.Will come back to this.
* Builder class also has methods as appName,master,config\(k,v\).
* So Fluent Interface is used to create a SparkSession object ie We have builder method inside SparkSession Companion object ,then when we call a builder method a Builder object is returned then we can use this to set various values using the setter method which sets the value\(using a HashMap ,options\) and returns the builder object ,finally when we call a getOrCreate method in builder object ,this will create a SparkSession object using the options HashMap.
* SparkSession needs a SparkContext , Option\[SharedState\], Option\[SessionState\] , SparkSessionExtensions.

## Main Entrance Point

When you submit a spark job ,the main entrance for the Job is **org.apache.spark.deploy.SparkSubmit**. This sets up the class path and other properties to be used by rest of the SparkCode written by users.

All the arguments that you pass while running the spark-submit ,they are sent as a Array of Strings to  SparkSubmitArguments class,this class extends SparkSubmitOptionParser which has all the arguments parse list and it makes sure only the allowed parameters are available.

Next Utils.getDefaultPropertiesFile is called which looks at env value SPARK\__CONF_\_DIR or SPARK\_HOME to get the absolute path of spark-defaults.conf,also  **sys.props Map** is loaded with these properties and also we get all the properties from the conf file loaded into a hashmap spark.properties Map.Here the SparkSubmitArguments  has master, executor,action etc different fields which are set .The priority is given to one that has been passed via command line ,next to spark-default.conf and next to default.

SparkSubmit -&gt; SparkSubmitArguments -&gt; SparkSubmitOptionParse

`if (master.startsWith("yarn")) {`

`val hasHadoopEnv = env.contains("HADOOP_CONF_DIR") || env.contains("YARN_CONF_DIR")`

`if (!hasHadoopEnv && !Utils.isTesting) {`

`throw new Exception(s"When running with master '$master' " +`

`"either HADOOP_CONF_DIR or YARN_CONF_DIR must be set in the environment.")`

`}`

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


