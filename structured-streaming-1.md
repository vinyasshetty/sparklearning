Now Spark Structured Streaming is quite different when compared to Spark Streaming DStreams.

So to better understand Structured Streaming,we need to put our Spark Streaming knowledge in the back burner and try to compare them only when required otherwise most of the time its better to think with a fresh mind towards Structured Streaming.

Structured Streaming provides a similar interface as that of a Batch Spark DataFrame/DataSet.Few of the source where spark can read from is File/Directories , Kafka, TwitterUtils,rate,socket.

```
val stream1 = spark.readStream
      .format("socket")
      .option("host", "localhost")
      .option("port", 5432)
      .load()

    val ds1 = stream1.as[String].flatMap(x => x.split("\\s+"))

    val cnts = ds1.groupBy($"value").count()
    //println(cnts.explain(true))

    val wt = cnts.writeStream
      .format("console")
      .outputMode("complete")
        .trigger(Trigger.ProcessingTime(10 seconds))
      .start()

    //println(wt.explain(true))
    wt.awaitTermination()
```



## Trigger.ProcessingTime

```
Trigger.ProcessingTime ==>
The query will be executed with micro-batches mode, where micro-batches will be kicked off at the 
user-specified intervals.

1)If the previous micro-batch completes within the interval, then the engine will wait until the 
interval is over before kicking off the next micro-batch.
2)If the previous micro-batch takes longer than the interval to complete (i.e. if an interval boundary 
is missed), then the next micro-batch will start as soon as the previous one completes (i.e., it will not wait for the next interval boundary).
3)If no new data is available, then no micro-batch will be kicked off.
```



A Simple example : **Streaming2\_Socket**

 Now the thing to note here is : Spark treats the input data as a table and new records are being added to them regularly.

Important to thing to note here is the "ouputMode" .It can be complete,update ,append.

By complete we mean that once the stream starts ,spark is gonna keep information of the output result and keep updating the info of the result table and every time it writes ,it will write the whole result table ie :

If i run the above code and then in terminal say as :

```
Vinyass-MacBook-Pro:~ vinyasshetty$ nc -lk 5432
vinyas shetty
hello vinyas

Then spark will give the result as :
-------------------------------------------
Batch: 0
-------------------------------------------
+------+-----+
| value|count|
+------+-----+
| hello|    1|
|vinyas|    2|
|shetty|    1|
+------+-----+

Now spark will not be trigerred again since it sees no change in data.
Also when it sees a chnage in data ,spark has a internal clock which keeps running and for every 
10 seconds it will see if there was change and if yes,then it will trigger again :

Now i will say as :
vinyas shetty <SAY ENTER>

spark will give result as :
-------------------------------------------
Batch: 1
-------------------------------------------
+------+-----+
| value|count|
+------+-----+
| hello|    1|
|vinyas|    3|
|shetty|    2|
+------+-----+
```

Two things to notice ,once count is getting updated and everytime you see the "complete" result being print.So this is the effect of complete mode. With that said ,**spark does NOT support non-aggregation operation in "complete" mode **because it will be massive data that spark needs to keep note of.

```
//Below will FAIL
val stream1 = spark.readStream
      .format("socket")
      .option("host", "localhost")
      .option("port", 5432)
      .load()

    val ds1 = stream1.as[String].flatMap(x => x.split("\\s+"))

    val cnts = ds1.map($"value") //Just a Dummy map without any aggregation

    val wt = cnts.writeStream
      .format("console")
      .outputMode("complete")  //If we chnage this to append it will work
        .trigger(Trigger.ProcessingTime(10 seconds))
      .start()

    wt.awaitTermination()
```

"append" mode feature:

Ex : **Streaming1\_Append**

```
 val stream1 = spark.readStream
      .format("socket")
      .option("host", "localhost")
      .option("port", 5432)
      .load()

    val ds1 = stream1.as[String].flatMap(x => x.split("\\s+"))

    val cnts = ds1.select($"value")
    //println(cnts.explain(true))

    val wt = cnts.writeStream
      .format("console")
      .outputMode("append")
      .trigger(Trigger.ProcessingTime(10 seconds))
      .start()

    wt.awaitTermination()
```

Input :

```
Vinyass-MacBook-Pro:~ vinyasshetty$ nc -lk 5432
vinyas shetty

Spark will give result as : 
-------------------------------------------
Batch: 0
-------------------------------------------
+------+
| value|
+------+
|vinyas|
|shetty|
+------+

Then i will add :
hello vinyas

-------------------------------------------
Batch: 1
-------------------------------------------
+------+
| value|
+------+
| hello|
|vinyas|
+------+

Now as you see spark will output only the latest record which reads from the input .And it does NOT
maintain history because of this we cannot do a regular aggregation(we have a workaround) 
like we did in complete mode
```

One more Example of append and aggregation\(**Streaming1\_1\_Append\)** :

```
 spark.sparkContext.setLogLevel("ERROR")
    val stream1 = spark.readStream
      .format("socket")
      .option("host", "localhost")
      .option("port", 5432)
      .load()

    val ds1 = stream1.as[String].flatMap(x => x.split("\\s+")).groupByKey(x=> x) 
    //Now the difference here is groupByKey returns a KeyValueGroupedDataset 
    //and NOT a RelationalGroupedDataset. If we have a dataset which has RelationalGroupedDataset ,then only
    //update and complete mode is supported.

    val cnts = ds1.mapGroups((k,i)=> (k,i.size))
    //println(cnts.explain(true))

    val wt = cnts.writeStream
      .format("console")
      .outputMode("append")
      .trigger(Trigger.ProcessingTime(10 seconds))
      .start()

    wt.awaitTermination()
```

Input :

```
Vinyass-MacBook-Pro:~ vinyasshetty$ nc -lk 5432
vinyas shetty

Spark will give :
-------------------------------------------
Batch: 0
-------------------------------------------
+------+---+
|    _1| _2|
+------+---+
|vinyas|  1|
|shetty|  1|
+------+---+

Then i feed input as :
hello vinyas <say enter>

then spark gives output as :
-------------------------------------------
Batch: 1
-------------------------------------------
+------+---+
|    _1| _2|
+------+---+
| hello|  1|
|vinyas|  1|
+------+---+

Now as you see the two important points ,count of vinyas did NOT increase in Batch 1 and also Batch 1 did
NOT have the shetty row of Batch 0.

If i feed input now as :
hello vinyas vinyas

-------------------------------------------
Batch: 2
-------------------------------------------
+------+---+
|    _1| _2|
+------+---+
| hello|  1|
|vinyas|  2|
+------+---+
 // This current processing data had vinyas twice.
```

"update" mode : this is a fusion of complete and append . ie it will behave same as "append" if there is no aggregation.But if there is a aggregation then it will be same as complete,but will ouput only the current result.Will be clear with below example:

**Streaming1\_1\_Update**

```
 spark.sparkContext.setLogLevel("ERROR")
    val stream1 = spark.readStream
      .format("socket")
      .option("host", "localhost")
      .option("port", 5432)
      .load()

    val ds1 = stream1.as[String].flatMap(x => x.split("\\s+")).groupBy($"value")

    val cnts = ds1.count()

    val wt = cnts.writeStream
      .format("console")
      .outputMode("update")
      .trigger(Trigger.ProcessingTime(10 seconds))
      .start()

    wt.awaitTermination()
```

 

```
1st input :
Vinyass-MacBook-Pro:~ vinyasshetty$ nc -lk 5432
vinyas shetty

Spark will output as :
-------------------------------------------
Batch: 0
-------------------------------------------
+------+-----+
| value|count|
+------+-----+
|vinyas|    1|
|shetty|    1|
+------+-----+

Then i will give input as :
hello vinyas < say enter>


Spark will output as :
-------------------------------------------
Batch: 1
-------------------------------------------
+------+-----+
| value|count|
+------+-----+
| hello|    1|
|vinyas|    2|
+------+-----+

Now as you see its a fusion ie count of vinyas is updated ,but the result in batch 1 will not have old data ie 
shetty record.

Now if i say:
shetty hey <enter>

-------------------------------------------
Batch: 2
-------------------------------------------
+------+-----+
| value|count|
+------+-----+
|   hey|    1|
|shetty|    2|
+------+-----+

Now shetty gets updated and the output will have records from the current data only.
```

E**x : Streaming\_File1**

Below i will show you a example of how the input source can be directory

```
 val spark = SparkSession.builder()
        .config("spark.sql.streaming.schemaInference","true")//Now not a good thing ,always good to give explicit schema
        .appName("file1").master("local[*]").getOrCreate()

  val fstream = spark.readStream
      .format("csv")
      .option("maxFilesPerTrigger",1) //Reads one file at a time in the directory.
      .option("header","true")
      //.option("inferSchema","true") //This does NOT WORK
      .option("latestFirst","true")
      .load("/Users/vinyasshetty/data_spark_structured_streaming/data/people/")

    println(fstream.printSchema())

    val fwstream = fstream.writeStream
      .format("console")
      .outputMode("append")
      .trigger(Trigger.ProcessingTime(10 seconds))
      .start()

    fwstream.awaitTermination()
```

Input :

```
Vinyass-MacBook-Pro:~ vinyasshetty$ ls -l /Users/vinyasshetty/data_spark_structured_streaming/data/people/
total 24
-rw-r--r--  1 vinyasshetty  staff  105 Mar 16 18:02 1.csv
-rw-r--r--  1 vinyasshetty  staff   75 Mar 16 18:02 2.csv
-rw-r--r--  1 vinyasshetty  staff   76 Mar 24 07:25 3.csv
Vinyass-MacBook-Pro:~ vinyasshetty$ cat /Users/vinyasshetty/data_spark_structured_streaming/data/people/*csv
name,city,country,age
Amy,Paris,FR,30
Bob,New York,US,22
Charlie,London,UK,35
Denise,San Francisco,US,22
name,city,country,age
Edward,London,UK,53
Francis,,FR,22
George,London,UK,
name,city,country,age
Vinyas,London,UK,53
Namratha,,FR,22
Varsha,London,UK,
Vinyass-MacBook-Pro:~ vinyasshetty$
```

Result :

```
root
 |-- name: string (nullable = true)
 |-- city: string (nullable = true)
 |-- country: string (nullable = true)
 |-- age: string (nullable = true)

()
-------------------------------------------
Batch: 0
-------------------------------------------
+--------+------+-------+----+
|    name|  city|country| age|
+--------+------+-------+----+
|  Vinyas|London|     UK|  53|
|Namratha|  null|     FR|  22|
|  Varsha|London|     UK|null|
+--------+------+-------+----+

-------------------------------------------
Batch: 1
-------------------------------------------
+-------+-------------+-------+---+
|   name|         city|country|age|
+-------+-------------+-------+---+
|    Amy|        Paris|     FR| 30|
|    Bob|     New York|     US| 22|
|Charlie|       London|     UK| 35|
| Denise|San Francisco|     US| 22|
+-------+-------------+-------+---+

-------------------------------------------
Batch: 2
-------------------------------------------
+-------+------+-------+----+
|   name|  city|country| age|
+-------+------+-------+----+
| Edward|London|     UK|  53|
|Francis|  null|     FR|  22|
| George|London|     UK|null|
+-------+------+-------+----+
```

Now as you see since we asked spark to infer  and it inferred all to be string.So its better to explicitly give a schema like how we do in batch.

For Testing purpose we can use "rate" ,this will help in generating quick datasets: **Streaming3\_Rate**

```
val rstream = spark.readStream
      .format("rate")
      .option("rowsPerSecond",100)
      .option("numPartitions",10)
      .load() // This is 

    println(rstream.isStreaming)
    //println(rstream.count()) //We cannot do a count??
    val cstream = rstream.groupBy().count()

    val wstream = cstream.writeStream
      .outputMode("complete")
      .format("console")
      .option("truncate","false")
      .trigger(Trigger.ProcessingTime(5 seconds)) // So each batch will have 500 records
      .start()

Try this(Use append mode and remove grouping if you want to view the actual data) :
rstream will be :
root
 |-- timestamp: timestamp (nullable = true)
 |-- value: long (nullable = true)
```

 

