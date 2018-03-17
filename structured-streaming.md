Now we will be learning Spark Structured Streaming where we will be using DataSet/DataFrame API for Streaming related transformation.

```
//We will create a SparkSession object as usual
val spark = SparkSession().builder.appName("test1").master("local[*]").getOrCreate()
//We will have a 
val stream1 = spark.readStream //DataStreamReader
                   .format("socket") //This can text,json,csv,jdbc,etc
                   .option("hostname","localhost")
                   .option("port",5401)
                   .load()

val words = stream1.as[String].flatMap(_.split("\\s+"))
val cnts = words.groupBy($"value").count() //Is this a action???

val res = cnts.writeStream // DataStreamWriter
              .format("console")
              .outputMode("complete")
              .start() // DataStreamWriter has a start method which will start the computation
              //Thing to note here is there is NO save method unlike DataFrameWriter,so to save the 
              //result on hdfs,use .option("path","<location>")

res.awaitTermination()
```

In Structured Streaming,we think of a input stream as a unbounded table ,where as a data comes it becomes appended as new rows to the unbounded table.

The transformation on the input will generate a "output result table" ,this output result table is wat will be written to the sink.

Now this output result table writing to sink can be handled in different Modes:

"Complete" =&gt; Entire updated result table will be written to sink.

"Append" =&gt; Only the new rows appended the the result table will be written to sink.It does NOT expect the existing result table values to change and hence this can be applied only on such queries.

"Update" =&gt; Here the result table will have only the records that have changed from the last time.The way this is different from "Complete" is this will have the information of only the updated records and this will behave same as "append" mode for non-aggregation operations.

Because of these above definitions,spark will put some limitations on the type of queries u can run on the input stream data.

Way it works is ,spark takes a existing unbounded input data table and does the transformation you have given and stores the result in a "result table" within itself.Now once we get new records on the input unbounded data table,spark will run transformation on the new records and based on the "output mode" it will aggregate the information with the existing result table.Now due to this spark sets some limitations like you cannot do a aggregation operation on a "append" mode,since as per append definition it cannot combine old result table with the new output.\[We will talk more\]

Some InBuilt Input Source :

**FileSources**\(csv,jsao,parquet,text,avro,etc\).Thing to note is file should be atomically placed ie say you place a file and stream reads it,then time you need a add a new file for the stream to read it,if you update the same file,stream will not read it.

```
val fstream = spark.readStream
                   .format("csv")
                   .option("maxFilesPerTrigger","2") //Two files read per trigger
                   .option("latestFirst","true") // default is false
                   .option("fileNameOnly","true") //default is false
                   .load("path")
```

Kafka : We will discuss this in detail ahead.

Socket : As see earlier .This is for testing only

Rate Source : This is only for testing ,creates a DataFrame with two columns \(timestamp and value\).

```
 val rstream = spark.readStream
      .format("rate")
      .option("rowsPerSecond",100)
      .option("numPartitions",10)
      .load()

    val wstream = rstream.writeStream
      .outputMode("append")
      .format("console")
      .option("truncate","false")
      .start()

    wstream.awaitTermination()

    +-----------------------+-----+
|timestamp              |value|
+-----------------------+-----+
|2018-03-17 09:04:21.046|300  |
|2018-03-17 09:04:21.056|301  |
|2018-03-17 09:04:21.066|302  |
|2018-03-17 09:04:21.076|303  |
|2018-03-17 09:04:21.086|304  |
|2018-03-17 09:04:21.096|305  |
```



