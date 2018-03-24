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
              .start() // DataStreamWriter has a start method which will start the computation.
              //This start method returns a StreamingQuery
              //Thing to note here is there is NO save method unlike DataFrameWriter,so to save the 
              //result on hdfs,use .option("path","<location>")

res.awaitTermination()
```

In Structured Streaming,we think of a input stream as a unbounded table ,where as a data comes it becomes appended as new rows to the unbounded table.

**start method on the DataStreamWriter returns a StreamingQuery object.This StreamingQuery is the one which continuosly keeps exceuting the queries.It also has the explain method.But to print the plan from explain ,it seems to expect data first.It seems to not work.**

The awaitTermination is on the StreamingQuery

The transformation on the input will generate a "output result table" ,this output result table is wat will be written to the sink.

Now this output result table writing to sink can be handled in different Modes:

"Complete" =&gt; Entire updated result table will be written to sink.This will maintian entire history of the result table.

"Append" =&gt; Only the new rows appended the the result table will be written to sink.It does NOT expect the existing result table values to change and hence this can be applied only on such queries,basically aggregataions involving count,agg metjods are NOT allowed.

"Update" =&gt; Here the result table will have only the records that have changed from the last time.The way this is different from "Complete" is this will have the information of only the updated records\(due to this sorting is NOT allowed since that would required all history info\) and this will behave same as "append" mode for non-aggregation operations.

Because of these above definitions,spark will put some limitations on the type of queries u can run on the input stream data.

Way it works is ,spark takes a existing unbounded input data table and does the transformation you have given and stores the result in a "result table" within itself.Now once we get new records on the input unbounded data table,spark will run transformation on the new records and based on the "output mode" it will aggregate the information with the existing result table.Now due to this spark sets some limitations like you cannot do a aggregation operation on a "append" mode,since as per append definition it cannot combine old result table with the new output.\[We will talk more\].

**Now we will have one Output result table per DataStreamWriter.This will result table information varies on output modes.if we we using complete mode,then the result table will maintain full history and it expects to have a key and a value ie a group by and aggregation operations only.**

**In append mode ,result table will have only the latest output data.**

Some InBuilt Input Source :

**FileSources**\(csv,jsao,parquet,text,avro,etc\).Thing to note is file should be atomically placed ie say you place a file and stream reads it,then time you need a add a new file for the stream to read it,if you update the same file,stream will not read it.

This behaves the same way as your static data file sources,but we have two extra features **latestFirst and maxFilesPerTrigger**

```
val fstream = spark.readStream
                   .format("csv")
                   .option("maxFilesPerTrigger","2") //Two files read per trigger
                   .option("latestFirst","true") // default is false
                   .option("fileNameOnly","true") //default is false
                   .load("path")
```

**Kafka** : We will discuss this in detail ahead.

**Socket **: As see earlier .This is for testing only

**Rate Source **: This is only for testing ,creates a DataFrame with two columns \(timestamp and value\).

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

These examples generate streaming DataFrames that are untyped, meaning that the schema of the DataFrame is not checked at compile time, only checked at runtime when the query is submitted. Some operations like `map`, `flatMap`, etc. need the type to be known at compile time. To do those, you can convert these untyped streaming DataFrames to typed streaming Datasets using the same methods as static DataFrame.

```
rstream.isStreaming => to check if this is a streaming dataframe/dataset
```

We can inferschema in Streaming,but it will make all columns as string type and its not rcommended:

```
  val spark = SparkSession.builder().config("spark.sql.streaming.schemaInference","true").appName("file1").master("local[*]").getOrCreate()
    spark.sparkContext.setLogLevel("ERROR")

//.option("inferSchema","true") on a DataStreamReader does NOt work.
```

Better Way to Do it :

```
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.catalyst.ScalaReflection
import org.apache.spark.sql.types.StructType
case class User(name:String,city:String,country:String,age:Int)
object St_Refelection_File {

  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder().appName("ref1").master("local[*]").getOrCreate()
    import spark.implicits._
    spark.sparkContext.setLogLevel("ERROR")


    val userschema = ScalaReflection.schemaFor[User].dataType.asInstanceOf[StructType]
    //For Reflection to work the User case class should NOt be nested   


    val rstream = spark.readStream
      .format("csv")
      .option("header",true)
      .option("maxFilesPerTrigger",1)
      .schema(userschema)
      .load("/Users/vinyasshetty/data_spark_structured_streaming/data/people/").as[User]

    println(rstream.printSchema())

    val wstream = rstream.writeStream.outputMode("append").format("console").start()

    wstream.awaitTermination(10000)
```



