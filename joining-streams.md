Joining Streams data with Static data is straight forward.Once you join the data ,all the things wrt Basics,window and watermarking is applicable based on output mode.

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





