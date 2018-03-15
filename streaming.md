Spark Streaming uses micro batch architecture  where we continuosuly receieve data and those are converted into micro-batches which is nothing but RDD's and on that we do processing.

Spark Streams is built on a abstraction called DStreams\(Discretized Stream\).A DStream is just a sequence of RDD's which are being processed one at a time\(group of RDD's can also be processed using window\).

DStream is created using a StreamingContext.

DStreams have transformations and actions\(there is a subtle diff,we will cover further\).

Streaming Context can be created in 3 types:

* StreamingContext\(conf:SparkConf,d:Duration\)
* StreamingContext\(master:String,appName:String,d:Duration\) //Have left out some optional
* StreamingContext\(sc:SparkContext\)

In the first two SparkContext is created internally within StreamingContext\(accessible via ssc.sparkContext\).I prefer the third one.

Now the way it works is when you submit your spark streaming job,we will have one task in the Executor which continuosuly keeps getting data from wherever the StreamingContext is connected to.

Now that task which start forming blocks/partitions at every spark.streaming.blockInterval\(default is 200ms\) duration and form  RDD at the duration given while creating the StreamingContext.

```
val ssc = StreamingContext(sc,Seconds(60)) 
//Now we will have a RDD created for every 60 seconds and the number of partitions 
//will be the data which has comes at every spark.streaming.blockInterval
```

StreamingContext at every spark.streaming.blockInterval duration informs the BlockManager process which replicates/persists this data on memory/disk,same way we used to persist/cache rdd ie like StorageLevel.MEMORY\_AND\_DISK\_SER\_2.

So remember two important times Block Interval\(set with spark.streaming.blockInterval\) and Batch Interval\(while creating StreamingContext\). MillisecondsSeconds ,Minutes are objects of Duration. Eg : Minutes\(1\).

We will have one task of the executor per dstream which will act as a receiver.

```
object StreamingPort1 {
  def main(args:Array[String])={
    val sc =  new SparkContext(new SparkConf().setMaster("local[*]").setAppName("Streaming1"))
    val ssc = new StreamingContext(sc,Seconds(10))
    val dstream1 = ssc.socketTextStream("localhost",5800,StorageLevel.MEMORY_AND_DISK_SER_2) 
    //When you create a dstream connection ,
    //at that time we can give the StorageLevel as to what sort of persisting u want on the block
    dstream1.map(x => x + "VIN").print(10)
    ssc.start()
    ssc.awaitTermination()
  }
```

When you say "start" on the StreamingContext is when the process will actually start running and if you have a action like print,saveAsFile,foreach then the corresponding transformation is run.Note "count" is NOT a action.

