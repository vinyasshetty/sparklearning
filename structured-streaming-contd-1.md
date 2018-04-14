DropDuplicates:

Spark keeps tracking of all the records and emits only one set of records,if the id had already come in the past ,then it will not output it again.

```
val df1 = spark.readStream.format("socket").option("host","localhost").option("port",5430).load()

  val df2 = df1.as[String].map(x=>x.split(","))

  val df3 = df2.select($"value"(0).as("name"),
    $"value"(1).cast(IntegerType).as("id"),
    $"value"(2).cast(TimestampType).as("ts")).dropDuplicates(Array("id"))

  val df4 = df3.writeStream
    .format("console")
    .option("truncate","false")
    .trigger(Trigger.ProcessingTime(10 second))
    .outputMode("append").start()
```

```
Vinyass-MacBook-Pro:~ vinyasshetty$ nc -lk 5430
juil,11,2018-03-17 09:04:33
ram,12,2018-03-17 09:05:12
ghy,21,2018-03-17 09:04:28
finch,34,2018-03-17 09:04:44
ghy,21,2018-03-17 09:04:28
juil,11,2018-03-17 09:04:33

-------------------------------------------
Batch: 0
-------------------------------------------
+-----+---+-------------------+
|name |id |ts                 |
+-----+---+-------------------+
|finch|34 |2018-03-17 09:04:44|
|ram  |12 |2018-03-17 09:05:12|
|ghy  |21 |2018-03-17 09:04:28|
|juil |11 |2018-03-17 09:04:33|
+-----+---+-------------------+


Next we input :
finch,34,2018-03-17 09:04:44
rini,8,2018-03-17 09:04:59
ghy,21,2018-03-17 09:04:55

We get only(Spark ignores id 34 and 21 ,since it had come earlier and spark keeps track of the result) :
-------------------------------------------
Batch: 3
-------------------------------------------
+----+---+-------------------+
|name|id |ts                 |
+----+---+-------------------+
|rini|8  |2018-03-17 09:04:59|
+----+---+-------------------+
```

**We can do append,complete,update.Same rules as earlier applies to the ouput mode.**

**We can also add withWatermark,this makes sure spark will remember the result for  that duration.**

**Like Join , dropDuplicates is not supported after aggregation on a streaming DataFrame/Dataset;**

### Unsupported Operations {#unsupported-operations}

1. NO distinct allowed
2. order by only after aggregation and  complete mode.
3. action like : count,foreach,show,limit not available.
4. Chained aggregations not allowed.

## DataStreamWriter :

```
df.writeStream //DataStreamWriter
.format("console") //
.queryName("uniquename")
.trigger(Trigger.ProcessingTime(10 second))
.ouputMode("") //append,complete,update
.option("checkpointLocation","<hdfs_path>")
.start() // Returns a Streaming Query
```

### Output modes:

As discussed earlier,we have complete,append and update mode.

Append : This is the default mode,we can use this for aggregations with watermark only,supports all other non -aggregation operations without watermark.Supports Stream-Stream Join ,along with condition we discussed about.

Complete : This is used only when we have aggregations.

Update: This is a combination of above .

## Sink:

File ,we write to hdfs but only with append mode.We can also use partitionBy,different formats and compressions.

```
df.writeStream
.format("parquet")
.option("path","")
.partitionBy($"",$"")
.start()
```

Kafka Sink \( All 3 output modes supported\):

```
df.writeStrea
.format("kafka")
.option("bootstrap.servers",",")
.option("enable.auto.commit","false")
.option("topic","")
.start()
```

Console Sink \( All 3 output modes supported\):

```
df.writeStream.
format("console")
.option("numRows",20)
.option("truncate","false")
.start()
```

Memory Mode\(Complete and Append Supported\)

Foreach\(All 3 modes\):

Some Details on Foreach :

```
  val df1 = spark.readStream.format("socket").option("host","localhost").option("port",5430).load()

  val df2 = df1.as[String]

  class Foreachimpl extends ForeachWriter[String] with Serializable{
    override def open(partitionId: Long, version: Long): Boolean = {
      println(s"Partition Id : ${partitionId} and Version : ${version}")
      true
    }

    override def process(value: String): Unit = {
      println(s"Vinyas == ${value}")
    }

    override def close(errorOrNull: Throwable): Unit = {
    }
  }

  val ob1 = new Foreachimpl()

  val sq = df2.writeStream.trigger(Trigger.ProcessingTime(20 second)).foreach(ob1).start()

  sq.awaitTermination()
```

I have implemented a ForeachWriter\[String\] since when i do a foreach on df2 ,i gets strings,if you use a Dataframe then you need to implement a ForeachWriter\[Row\].

open method is run when a trigger  is done and for every trigger its runs for each partition.Below as you see, by default 4 partitions are created for my input data.Now version is zero since this is run from the first trigger.this method returns a boolean,now if it returns a boolean for a given partition,then below process method is NOT called for that partition.

process method is run for every record in the partition

```
Vinyass-MacBook-Pro:~ vinyasshetty$ nc -lk 5430
vinyas,1,2018-03-17 09:04:21
namratha,2,2018-03-17 09:04:23
varsha,3,2018-03-17 09:04:33

Output:
Partition Id : 0 and Version : 0
Partition Id : 2 and Version : 0
Partition Id : 3 and Version : 0
Partition Id : 1 and Version : 0
Vinyas == vinyas,1,2018-03-17 09:04:21
Vinyas == namratha,2,2018-03-17 09:04:23
Vinyas == varsha,3,2018-03-17 09:04:33

Now i send input :
vinyas,1,2018-03-17 09:04:21
shetty,2,2018-03-17 09:04:23

Ouput (Version is increased):
Partition Id : 2 and Version : 1
Partition Id : 0 and Version : 1
Partition Id : 1 and Version : 1
Vinyas == vinyas,1,2018-03-17 09:04:21
Partition Id : 3 and Version : 1
Vinyas == shetty,2,2018-03-17 09:04:23
```

See below example:

```
class Foreachimpl extends ForeachWriter[String] with Serializable{
    override def open(partitionId: Long, version: Long): Boolean = {
      println(s"Partition Id : ${partitionId} and Version : ${version}")
      if(partitionId == 1)
      true
      else false
    }
    override def process(value: String): Unit = {
      println(s"Vinyas == ${value}")
    }

   override def close(errorOrNull: Throwable): Unit = {
    }
  }
  val ob1 = new Foreachimpl()

  val sq = df2.writeStream.trigger(Trigger.ProcessingTime(20 second)).foreach(ob1).start()

  sq.awaitTermination()

  Input:
  Vinyass-MacBook-Pro:~ vinyasshetty$ nc -lk 5430
vinyas,1,2018-03-17 09:04:21
namratha,2,2018-03-17 09:04:23
varsha,3,2018-03-17 09:04:33
vinyas,1,2018-03-17 09:04:21

Partition Id : 3 and Version : 0
Partition Id : 2 and Version : 0
Partition Id : 1 and Version : 0
Partition Id : 0 and Version : 0
Vinyas == namratha,2,2018-03-17 09:04:23

Now as you see if we have 4 partitions ,so 4 instances of Foreachimpl is created(These instances are created
in driver itself) and Each instance is run in a task in the executor(ie pen,process,close is run in executors).
First open method is run and for that given partition if it gives a true,then corresponding process method 
is run for that partition and finally the close method.In above example,we return true only on Partition 1,
so only one record is written since other records seems to have been put into partition 3,2,1 and we 
returned false for them in open,so process was not called for those partitions.

Important Point about close method from Spark docs:
Whenever open is called, close will also be called (unless the JVM exits due to some error). 
This is true even if open returns false. If there is any error in processing and writing the data,
 close will be called with the error. It is your responsibility to clean up state (e.g. connections, 
 transactions, etc.) that have been created in open such that there are no resource leaks.

```



