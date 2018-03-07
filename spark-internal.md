* SparkSession is a companion object which has the builder method which returns a Builder Object,.
* Builder has fields : options which is a mutable HashMap\[String,String\] ,  extension:SparkExtensions object , userSuppliedContext:Option\[SparkContext\] which set to None
* SparkExtensions =&gt; This seems to be the the place for all the Rules Builders for different Plans.Will come back to this.
* Builder class also has methods as appName,master,config\(k,v\).
* So Fluent Interface is used to create a SparkSession object ie We have builder method inside SparkSession Companion object ,then when we call a builder method a Builder object is returned then we can use this to set various values using the setter method which sets the value\(using a HashMap ,options\) and returns the builder object ,finally when we call a getOrCreate method in builder object ,this will create a SparkSession object using the options HashMap.
* SparkSession needs a SparkContext , Option\[SharedState\], Option\[SessionState\] , SparkSessionExtensions.

## Main Entrance Point

spark-submit shell script calls spark-class shell script:

`if [ -z "${SPARK_HOME}" ]; then`

``export SPARK_HOME="$(cd "`dirname "$0"`"/..; pwd)"``

`fi`

`exec "${SPARK_HOME}"/bin/spark-class org.apache.spark.deploy.SparkSubmit "$@"`

spak-class gets the java jar\(RUNNER\),spark jars\(LAUNCH\_\_CLASSPATH\) ,hadoop jars\(HADOOP\_\_LZO\_JARS\).Then runs below :

`"$RUNNER" -Xmx128m -cp "$LAUNCH_CLASSPATH" org.apache.spark.launcher.Main "$@"`

org.apache.spark.launcher.Main is a java class where the classname is set to agrs\(0\) ie org.apache.spark.deploy.SparkSubmit.

Now `org.apache.spark.launcher.Main will create a Spark Running Command and return that to a variable in spark-class which is exceuted ,to view the command set`SPARK\_PRINT\_LAUNCH\_COMMAND=&lt;somevalue&gt; in the shell.

`boolean printLaunchCommand = !isEmpty(System.getenv("SPARK_PRINT_LAUNCH_COMMAND"));`

`if (printLaunchCommand) {`

`System.err.println("Spark Command: " + join(" ", cmd));`

`System.err.println("========================================");`

`}`

If SPARK\_PRINT\_LAUNCH\_COMMAND env variable is set then it will print your "actual running command".

When you submit a spark job ,the main entrance for the Job is **org.apache.spark.deploy.SparkSubmit**. This sets up the class path and other properties to be used by rest of the SparkCode written by users.

In **SparkSubmit main method , **SparkSubmitArguments  object is created.** **All the arguments that you pass while running the spark-submit ,they are sent as a Array of Strings to  SparkSubmitArguments class,this class extends SparkSubmitOptionParser which has all the arguments parse list and it makes sure only the allowed parameters are available.

Next Utils.getDefaultPropertiesFile is called which looks at env value SPARK\__CONF_\_DIR or SPARK\_HOME to get the absolute path of spark-defaults.conf,also  **sys.props Map** is loaded with these properties and also we get all the properties from the conf file loaded into a hashmap spark.properties Map\(Anything that has been passed via --conf in command line will ahve preference over conf file and env\).Here the SparkSubmitArguments  has master, executor,action etc different fields which are set .The priority is given to one that has been passed via command line ,next to spark-default.conf and next to default.

SparkSubmit -&gt; SparkSubmitArguments -&gt; SparkSubmitOptionParse.

Below are the values/fields that are set in SparkSubmitArguments class based on the command line,spark-default.conf and env variables.\(below i have given only the initial defaults\) .\`

` ` ` `

var master: String = null

var deployMode: String = null

var executorMemory: String = null

var executorCores: String = null

var totalExecutorCores: String = null

var propertiesFile: String = null

var driverMemory: String = null

var driverExtraClassPath: String = null

var driverExtraLibraryPath: String = null

var driverExtraJavaOptions: String = null

var queue: String = null

var numExecutors: String = null

var files: String = null

var archives: String = null

var mainClass: String = null

var primaryResource: String = null

var name: String = null

var childArgs: ArrayBuffer\[String\] = new ArrayBuffer\[String\]\(\)

var jars: String = null

var packages: String = null

var repositories: String = null

var ivyRepoPath: String = null

var packagesExclusions: String = null

var verbose: Boolean = false

var isPython: Boolean = false

var pyFiles: String = null

var isR: Boolean = false

var action: SparkSubmitAction = null

val sparkProperties: HashMap\[String, String\] = new HashMap\[String, String\]\(\)

var proxyUser: String = null

var principal: String = null

var keytab: String = null

// Standalone cluster mode only

var supervise: Boolean = false

var driverCores: String = null

var submissionToKill: String = null

var submissionToRequestStatusFor: String = null

var useRest: Boolean = true // used internally

` ` ` `

Then SparkSubmit  calls the submit method, which prepares the envt by calling :

`val (childArgs, childClasspath, sparkConf, childMainClass) = prepareSubmitEnvironment(args:SparkSubmitArguments)`

Then it calls :

`runMain(childArgs, childClasspath, sparkConf, childMainClass, args.verbose)`

Some Main parts of runMain Method :

`private def runMain(`

`childArgs: Seq[String],`

`childClasspath: Seq[String],`

`sparkConf: SparkConf,`

`childMainClass: String,`

`verbose: Boolean): Unit = {`

`// scalastyle:off println`

`if (verbose) {`

`printStream.println(s"Main class:\n$childMainClass")`

`printStream.println(s"Arguments:\n${childArgs.mkString("\n")}")`

`// sysProps may contain sensitive information, so redact before printing`

`printStream.println(s"Spark config:\n${Utils.redact(sparkConf.getAll.toMap).mkString("\n")}")`

`printStream.println(s"Classpath elements:\n${childClasspath.mkString("\n")}")`

`printStream.println("\n")`

`}`

`val app: SparkApplication = if (classOf[SparkApplication].isAssignableFrom(mainClass)) {`

`mainClass.newInstance().asInstanceOf[SparkApplication]`

`} else {`

`// SPARK-4170`

`if (classOf[scala.App].isAssignableFrom(mainClass)) {`

`printWarning("Subclasses of scala.App may not work correctly. Use a main() method instead.")`

`}`

`new JavaMainApplication(mainClass)`

`}`

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


