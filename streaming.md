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

 StreamingContext at every spark.streaming.blockInterval duration informs the BlockManager process which replicates this data on memory ussually the StorageLevel is MEMORY\_

