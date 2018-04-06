Joining Streams data with Static data is straight forward.Once you join the data ,on the joined data all the things wrt Basics,window and watermarking is applicable based on output modes same as discussed earlier.

**StreamStaticJoin1**

```
 case class Statdata(name:String,magic:Int,ts:java.sql.Timestamp)

  val statdata = Statdata("VIN",1,java.sql.Timestamp.valueOf("2018-03-17 09:04:21")) ::
    Statdata("Nam",2,java.sql.Timestamp.valueOf("2018-03-17 09:04:21")) :: Nil

  val df1 = spark.readStream.format("socket").option("host","localhost").option("port","5431").load()

  val df2 = df1.as[String].map(x=>x.split(","))

  val df3 = df2.select($"value"(0).as("name"),$"value"(1).cast(IntegerType).as("id"),$"value"(2).cast(TimestampType).as("ts"))

  val stat = spark.createDataFrame(statdata)

  val jdf = df3.join(stat,$"id" <=> $"magic","left_outer")
             .select($"id",df3("name"),stat("name"),df3("ts"),stat("ts"))
    .groupBy($"id").count()


    val df4 = jdf.writeStream.format("console")
    .trigger(Trigger.ProcessingTime(5 seconds)).outputMode("complete")
    .option("truncate","false").start()
```

Now "inner_join" is allowed ,regarding outerjoins ,leftouter is allowed when stream is in the and rightouter is allowed when stream is the right. otherwise spark will not know when to put nulls._

## Stream Stream Join

Starting from Spark 2.3 ,We can join two streams. **Stream-Stream join currently supports only append mode only.**

Inner Join without any watermarking ** StreamtToStream1 **:

```
val df1 = spark.readStream.format("socket").option("host","localhost").option("port","5431").load()

  val df2 = df1.as[String].map(x=>x.split(","))

  val df3 = df2.select($"value"(0).as("name"),$"value"(1).cast(IntegerType).as("id"),$"value"(2).cast(TimestampType).as("ts"))

  val df1_1 = spark.readStream.format("socket").option("host","localhost").option("port","5430").load()

  val df2_1 = df1_1.as[String].map(x=>x.split(","))

  val df3_1 = df2_1.select($"value"(0).as("name"),$"value"(1).cast(IntegerType).as("id"),$"value"(2).cast(TimestampType).as("ts"))

  val joindf = df3.join(df3_1,df3("id") <=> df3_1("id"))

  val res = joindf.writeStream.outputMode("append").trigger(Trigger.ProcessingTime(5 seconds))
    .format("console").option("truncate","false").start()

```







