* Spark has two main parts : Driver and Executors.Driver maintains the Spark jobs status ,request for resources from yarn and schedules different jobs. Executors are the ones where the actual tasks run on the underlying data and they report back the result to driver.

* Partition corresponds to one task in Spark..This is the basic unit of Parallelism.

* Partition is collection of data/rows on a single physical machine.A dataframe can have zero or more partitions.

* Spark core data structures ie RDD Dataframe and DataSet are immutable.

* A Rdd without partition ,in that data is assigned based on partition size and data size.

* A Rdd with partition will gurantee that either a value beloging to a given will always end up same partition or a range of keys will always go to a same partition.

* ** transformations with narrow dependencies as those in which “each partition of the parent RDD is used by at most one partition of the child RDD.” The creators define transformations with wide dependencies as transformations in which “multiple child partitions may depend on \[each partition in the parent\].” **

* Within a given stage the number of tasks used to complete a computation corresponds to each output partition rather than each input partition—when RDDs are evaluated; the tasks needed to compute a transformation are computed on the child partitions. ie say if i have a multi step narrow transformations where input/first-parent partition count is 200 and then a coalesce\(to 50\) and a action. Now this stage will run with just 50 tasks and not 200.

* master can be given to run on local mode with avialable cores "local\[\*\]" .

* Usually in Prod master is set to "yarn" ie if using yarn cluster manager.

* deploy-mode is either client or cluster.

* Preference of Config from Highest to Lowest : What has set in your individual application code , what has been set in your while you are running your jar via spark-submit ,what has been set on the cluster level.

* **spark.eventLog.dir** is to generate logs and a subdirectory is created for each application\(Keep this some hdfs location\) while **spark.history.fs.logDirectory** is the place where Spark History Server finds log events.

* To understand Complex DataTypes in Spark,Read the Accepted Answer  =&gt; [https://stackoverflow.com/questions/28332494/querying-spark-sql-dataframe-with-complex-types](https://stackoverflow.com/questions/28332494/querying-spark-sql-dataframe-with-complex-types)

To change logging information while running spark :

```
--conf "spark.driver.extraJavaOptions=-Dlog4j.configuration=log4j-spark.properties" 
--conf "spark.executor.extraJavaOptions=-Dlog4j.configuration=log4j-spark.properties"
```

We can put the properties file in the resources folder while building the jar else make sure properties file is in the classpath.

```
case class Am(x:List[Int]){
def recurs(a:List[Int],f1:Int=>Int):List[Int]={
a match {
case Nil => Nil
case y::ys => f1(y) :: recurs(ys,f1)
}
}
def map1(f:Int => Int)={
val l = recurs(x,f)
Am(l)
}
}
```

```
Handle with case class with has more then 22.Only when u canse scala 2.10 or older
class Demo(val field1: String,
    val field2: Int,
    // .. and so on ..
    val field23: String)

extends Product 
//For Spark it has to be Serializable
with Serializable {
    def canEqual(that: Any) = that.isInstanceOf[Demo]

    def productArity = 23 // number of columns

    def productElement(idx: Int) = idx match {
        case 0 => field1
        case 1 => field2
        // .. and so on ..
        case 22 => field23
    }
}
```

In the code when SparkContext is created that starts a driver and the corresponding executors.Each executor is its own JVM.

# **Schema**

StructType is similar to case class.

Structtype consists of list of StructFields.

Each StructField represents a column.

Schema is eagerly evaluated on dataframe transformations.See below

    scala> df1.select($"id1")
    org.apache.spark.sql.AnalysisException: cannot resolve '`id1`' given input columns: [id, attributes, zip, pt, happy];;
    'Project ['id1]
    +- LocalRelation [id#16L, zip#17, pt#18, happy#19, attributes#20]



###### The number of tasks that run depends on the number of partitions of the child rdd/dataframe.So if you have like below :

```
scala> df2.rdd.partitions.length
res48: Int = 30

scala> val df3 = df2.filter($"_1" > 2).select($"_1" + 10).coalesce(10)
df3: org.apache.spark.sql.Dataset[org.apache.spark.sql.Row] = [(_1 + 10): int]

scala> df3.count
res49: Long = 1

scala> df3.rdd.partitions.length
res50: Int = 10


```

So as you see above the number of partition the filter and map runs on is 10 instead of 20. Check the Spark UI  for the count job and you will see 10 tasks that would have run



