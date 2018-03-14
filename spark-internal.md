
---

---

## Main Entrance Point When We Do a Spark-Submit

spark-submit shell script calls spark-class shell script:

    if [ -z "${SPARK_HOME}" ]; then
    export SPARK_HOME="$(cd "`dirname "$0"`"/..; pwd)"
    fi
    exec "${SPARK_HOME}"/bin/spark-class org.apache.spark.deploy.SparkSubmit "$@"

spark-class shell gets the java jar\(RUNNER\),spark jars\(LAUNCH\_\_CLASSPATH\) ,hadoop jars\(HADOOP\_\_LZO\_JARS\).Then runs below :

`"$RUNNER" -Xmx128m -cp "$LAUNCH_CLASSPATH" org.apache.spark.launcher.Main "$@"`

org.apache.spark.launcher.Main is a java class where the classname is set to agrs\(0\) ie org.apache.spark.deploy.SparkSubmit.

Now `org.apache.spark.launcher.Main will create a Spark Running Command and return that to a variable in spark-class shell which is executed ,to view the command set`SPARK\_PRINT\_LAUNCH\_COMMAND=&lt;somevalue&gt; in the shell.

```
boolean printLaunchCommand = !isEmpty(System.getenv("SPARK_PRINT_LAUNCH_COMMAND"));
if (printLaunchCommand) {
System.err.println("Spark Command: " + join(" ", cmd));
System.err.println("========================================");
}
```

If SPARK\_PRINT\_LAUNCH\_COMMAND env variable is set then it will print your "actual running command".See below.Now this below command is executed by spark-class shell script.

```
Spark Command: /usr/jdk64/jdk1.7.0_67/bin/java -Dhdp.version=2.5.3.0-37 -cp /usr/hdp/current/spark2-client/conf/:/usr/hdp/current/spark2-client/jars/*:/usr/hdp/current/hadoop-client/conf/ -XX:MaxPermSize=256m org.apache.spark.deploy.SparkSubmit --master yarn --deploy-mode cluster --class com.spark2.viny.Spark1 sparklearning-1.0-SNAPSHOT.jar
```

As seen above ,when you submit a spark job ,the main entrance for the Job is **org.apache.spark.deploy.SparkSubmit**. This sets up the class path and other properties to be used by rest of the SparkCode written by users.

In **SparkSubmit main method , **SparkSubmitArguments  object is created.** **All the arguments that you pass while running the spark-submit ,they are sent as a Array of Strings to  SparkSubmitArguments class,this class extends SparkSubmitOptionParser which has all the arguments parse list and it makes sure only the allowed parameters are available.As part of this sparkproperties hashmap is set with values coming from command line --conf.Also Here the SparkSubmitArguments  has master, executor,action etc different fields which are set.

Next Utils.getDefaultPropertiesFile is called which looks at env value SPARK\__CONF_\_DIR or SPARK\_HOME/conf to get the absolute path of **spark-defaults.conf**. If the conf file is NOT there it does not throw a error.The priority is given to one that has been passed via command line ,next to spark-default.conf and next to default enviornment variable. Also any keys not starting with "spark." is removed.

SparkSubmit -&gt; SparkSubmitArguments -&gt; SparkSubmitOptionParse.

Below are the values/fields that are set in SparkSubmitArguments class based on the command line,spark-default.conf and env variables.\(below i have given only the initial defaults\) .

```
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
var childArgs: ArrayBuffer[String] = new ArrayBuffer[String]()
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
val sparkProperties: HashMap[String, String] = new HashMap[String, String]()
var proxyUser: String = null
var principal: String = null
var keytab: String = null
// Standalone cluster mode only
var supervise: Boolean = false
var driverCores: String = null
var submissionToKill: String = null
var submissionToRequestStatusFor: String = null
var useRest: Boolean = true // used internally
```

action can be "submit","kill" or "request\__status" .Default is "submit". ** Yarn does NOT support "kill" and "request\_status" **_

master = Option\(master\).getOrElse\("local\[\*\]"\) // ie if master is not set anywhere\(default conf,command line,env\) then it uses .

Then SparkSubmit  calls the submit method, which prepares the envt by calling :

`val (childArgs, childClasspath, sparkConf, childMainClass) = prepareSubmitEnvironment(args:SparkSubmitArguments)`

This SparkConf created in `prepareSubmitEnvironment` still does NOT have access to the conf created by user in the code.This will take command line,spark-defaults conf file  in that order.This uses the sparkproperties hashmap from SparkSubmitArguments.

```
 private[deploy] def prepareSubmitEnvironment(
      args: SparkSubmitArguments,
      conf: Option[HadoopConfiguration] = None)
      : (Seq[String], Seq[String], SparkConf, String) = {
    // Return values
    val childArgs = new ArrayBuffer[String]()
    val childClasspath = new ArrayBuffer[String]()
    val sparkConf = new SparkConf()
    var childMainClass = ""

    //Also make sure of below
    if (master.startsWith("yarn")) {
val hasHadoopEnv = env.contains("HADOOP_CONF_DIR") || env.contains("YARN_CONF_DIR")
if (!hasHadoopEnv && !Utils.isTesting) {
throw new Exception(s"When running with master '$master' " +
"either HADOOP_CONF_DIR or YARN_CONF_DIR must be set in the environment.")
}
```

In the above method based on master and deploy-mode ,we will decide on the childMainClass to run,for yarn cluster ,**childMainClass  is chosen as org.apache.spark.deploy.yarn.YarnClusterApplication**\(these are under resource-manager project\) and then childArgs will have Array\("--jars","vin.jar","--class","com.vin.Ex1","arg1"\) childClassPath will have all the required jars

Then it calls :

`runMain(childArgs, childClasspath, sparkConf, childMainClass, args.verbose)`

Some Main parts of runMain Method :

```
private def runMain(
childArgs: Seq[String],
childClasspath: Seq[String],
sparkConf: SparkConf,
childMainClass: String,
verbose: Boolean): Unit = {
// scalastyle:off println
if (verbose) {
printStream.println(s"Main class:\n$childMainClass")
printStream.println(s"Arguments:\n${childArgs.mkString("\n")}")
// sysProps may contain sensitive information, so redact before printing
printStream.println(s"Spark config:\n${Utils.redact(sparkConf.getAll.toMap).mkString("\n")}")
printStream.println(s"Classpath elements:\n${childClasspath.mkString("\n")}")
printStream.println("\n")
}

 mainClass = Utils.classForName(childMainClass)  // This just does the Class.forName(childMainClass)

 val app: SparkApplication = if (classOf[SparkApplication].isAssignableFrom(mainClass)) {
mainClass.newInstance().asInstanceOf[SparkApplication] 
} else {
// SPARK-4170
if (classOf[scala.App].isAssignableFrom(mainClass)) {
printWarning("Subclasses of scala.App may not work correctly. Use a main() method instead.")
}
new JavaMainApplication(mainClass)
}
  try {
      app.start(childArgs.toArray, sparkConf)
    } catch {
      case t: Throwable =>
        findCause(t) match {
          case SparkUserAppException(exitCode) =>
            System.exit(exitCode)

          case t: Throwable =>
            throw t
        }
    }
}
```

```
private[spark] class YarnClusterApplication extends SparkApplication {

  override def start(args: Array[String], conf: SparkConf): Unit = {
    // SparkSubmit would use yarn cache to distribute files & jars in yarn mode,
    // so remove them from sparkConf here for yarn mode.
    conf.remove("spark.jars")
    conf.remove("spark.files")

    new Client(new ClientArguments(args), conf).run()
  }

}
```

Point to note here is : we create a instance from mainClass ie org.apache.spark.deploy.yarn.YarnClusterApplication  which extends trait org.apache.spark.deploy.SparkApplication and has to implement method start which creates a org.apache.spark.deploy.yarn.Client object and calls run method on it. YarnClusterApplication and Client are in resource-managers project. ClientArguments method takes the array Array\("--jars","vin.jar","--class","com.vin.Ex1","arg1"\) and parses it accordingly to make them into fields of ClientArguments object as below :

```
private[spark] class ClientArguments(args: Array[String]) {

  var userJar: String = null
  var userClass: String = null
  var primaryPyFile: String = null
  var primaryRFile: String = null
  var userArgs: ArrayBuffer[String] = new ArrayBuffer[String]()

  parseArgs(args.toList)

  private def parseArgs(inputArgs: List[String]): Unit = {
    var args = inputArgs

    while (!args.isEmpty) {
      args match {
        case ("--jar") :: value :: tail =>
          userJar = value
          args = tail

        case ("--class") :: value :: tail =>
          userClass = value
          args = tail

        case ("--primary-py-file") :: value :: tail =>
          primaryPyFile = value
          args = tail

        case ("--primary-r-file") :: value :: tail =>
          primaryRFile = value
          args = tail

        case ("--arg") :: value :: tail =>
          userArgs += value
          args = tail

        case Nil =>

        case _ =>
          throw new IllegalArgumentException(getUsageMessage(args))
      }
    }
```

```
private[spark] class Client(
    val args: ClientArguments,
    val sparkConf: SparkConf)
  extends Logging {

  import Client._
  import YarnSparkHadoopUtil._

  private val yarnClient = YarnClient.createYarnClient
  private val hadoopConf = new YarnConfiguration(SparkHadoopUtil.newConfiguration(sparkConf))
 private val fireAndForget = isClusterMode && !sparkConf.get(WAIT_FOR_APP_COMPLETION)


/**
   * Submit an application to the ResourceManager.
   * If set spark.yarn.submit.waitAppCompletion to true, it will stay alive.Default is true ,what this means is 
   * when you submit a spark job ,the client shell will wait until the job is completed and it keeps reporting 
   * the status of the job in the client shell.
   * reporting the application's status until the application has exited for any reason.
   * Otherwise, the client process will exit after submission.
   * If the application finishes with a failed, killed, or undefined status,
   * throw an appropriate SparkException.
   */
   def run(): Unit = {
   /*
   * Get a new application from our RM
   * Verify whether the cluster has enough resources for our AM
   * Set up the appropriate contexts to launch our AM
   * Finally, submit and monitor the application.
   * In this the ApplicationMaster Launches the User Class Main method along with 
   * any arguments required by User class
   */
    this.appId = submitApplication()
    if (!launcherBackend.isConnected() && fireAndForget) {  // This is what decides whether a client should be active 
                                                            // or should it just be fire and forget
      val report = getApplicationReport(appId)
      val state = report.getYarnApplicationState
      logInfo(s"Application report for $appId (state: $state)")
      logInfo(formatReportDetails(report))
      if (state == YarnApplicationState.FAILED || state == YarnApplicationState.KILLED) {
        throw new SparkException(s"Application $appId finished with status: $state")
      }
    } else {
      /*
      * This below will go into a loop in monitorApplication method and keep monitoring the status of the application
      * from client shell.
      */
      val (yarnApplicationState, finalApplicationStatus) = monitorApplication(appId)
      if (yarnApplicationState == YarnApplicationState.FAILED ||
        finalApplicationStatus == FinalApplicationStatus.FAILED) {
        throw new SparkException(s"Application $appId finished with failed status")
      }
      if (yarnApplicationState == YarnApplicationState.KILLED ||
        finalApplicationStatus == FinalApplicationStatus.KILLED) {
        throw new SparkException(s"Application $appId is killed")
      }
      if (finalApplicationStatus == FinalApplicationStatus.UNDEFINED) {
        throw new SparkException(s"The final status of application $appId is undefined")
      }
    }
  }
```

_**Under resource-managers project ,we have yarn related spark configurations and they stored in org.apache.spark.deploy.yarn.config object.**_

How Spark Application Id is set in Yarn Mode:

```
 val newApp = yarnClient.createApplication()
      val newAppResponse = newApp.getNewApplicationResponse()
      appId = newAppResponse.getApplicationId()
```

org.apache.spark.yarn.deploy.ApplicationMaster creates a new thread on which it runs the user main method and then blocks\(join\) until this thread is done**&lt;NEED TO add more details around Yarn creation of application master etc =&gt; run,runImpl,runDriver,startUserApplication &gt;. Yarn\(Application Manager\) seems to be running/calling the companion object ApplicationMaster main method which in turns creates the ApplicationMaster object and call the run method on it .Need to confirm on this.**

```

```

## 

## 



