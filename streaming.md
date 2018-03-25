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

When you say "start" on the StreamingContext is when the process will actually start running and if you have a action like print,saveAsFile,foreach then the corresponding transformation is run.Note "count" is NOT a action.awaitTeramination basically makes sure that process does not while and is always active for streams of data to process.

Now point to make sure is one the first 10 Seconds is done,then a complete one RDD is formed and all the corresponding transformation and action you have created will run,when this is happening the receiver will continue to get the data and creating RDD,but the new RDD transformation/action is NOT triggered until the existing RDD is done with its transformation/action.

So best practice is to make sure to complete your process within the "batch duration".

Now sometimes you may want to run your process on a combinations of RDD's instead of just one for that we can use windows:

```
val dstream2 = dstream1.window(Seconds(300)) 
dstream2.<some transformation>
dstream2.<some action>
```

Now say our batch duration was set to Seconds\(60\), so we will one RDD per 60 seconds and dstream2 will end up having 5 RDD's and on that trans/action will be done.One problem here is your trans/action on dstream2 will run every 5 minutes but if you want it to run more frequently\(say 2 minutes\) but with last 5 RDD's\(5 minutes\) worth data the say :

```
val dstream2 = dstream1.window(windowDuration=Seconds(300),slideDuration=Seconds(120)) 
dstream2.<some transformation>
dstream2.<some action>

//PS: First time it will run with 2 minutes worth of data and second time it will run with 4 minutes 
//worth of data(Since 5 minutes worth of data is still not avialable ,
//but after that it will run every 2 minutes with last 5 minutes worth of data.
```

Now you may wonder why we need windowDuration and slideDuration and why not just adjust the "batch duration" .Well we can do it but if say we multiple processing worth differnt times to be done then we can have one main batch duration as 1 minute and then mutiple dstreams with window duration of 5 ,6 ,7 etc . Also longer we keep the batch duration its more risker of losing data due to failure\(well spark can recover,we will ahead fault tolerance.But you lose precious time\).

**One thing to note here is windowDuration and slideDuration should always be a multiple of batch duration.And Batch duration should be mutiple of block interval.Also batch duration cannot be Seconds\(6.5\) ,the value u send should be a long.**

Most of the ByKey and some non ByKey operators on dstreams provide in support for sending windowDuration and slideDuration.

```
dstream1.reduceByKeyAndWindow((x,y)=>x+y,Seconds(10),Seconds(2))
```

We can mutiple input initial dstreams created .Now we will have two tasks which will act as receivers and will keep receiving data and this data will be unionized.Make Sure dstreams which we union has the same type and also the same slide duration

```
object UnionDStream {
  def main(args:Array[String])={
    val sc = new SparkContext(new SparkConf().setMaster("local[*]").setAppName("UnionStream"))
    val ssc = new StreamingContext(sc,Seconds(20))

    val dstream1 = ssc.socketTextStream("localhost",5400)
    val dstream2 = ssc.socketTextStream("localhost",5401)

    val dstream3 = ssc.union(Array(dstream1,dstream2))

    dstream3.map(x => x + "VIN").print(10)

    ssc.start()
    ssc.awaitTermination()

  }

}
```

How do we select a good batch interval or a block interval .The number of tasks that can run on our cluster can be equal to number of cores \* number of executors.

batch interval = \(number of cores \* number of executors\) / \(\(number of receivers\) \*\(block interval\)\)

ie say ,our cluster can handle 12 tasks in parallel and we plan to run two recievers ,with block interval 1 seconds\(this is very high,just a example\),then our batch interval is 6 seconds.

"Stateless" : Usually the dstream operations which are NOt using window are called stateless because they do NOT depend on previous or future RDD's.We will talk later about State Based dstream operator.

StreamingContext has different states when it executes:

1. Initialized : This is the initial state of the streamingContext,the control has not yet reached ssc.start\(\) and we can still add new dstreams, add transformations/actions.
2. Active : Now the ssc.start\(\) is executed ,then StreamingContext goes into Active State and no new dstreams,transformations or actions can be added,in Active state only dstream based transformation,action is executed.
3. Stopped : Now ssc is stopped .

```
ssc.stop() => Stops the StreamingContext and SparkContext(even if its created outsideStreming Context.
ssc.stop(stopSparkContext=false)
ssc.stop(stopSparkContext=false,stopGracefully=true) //stops gracefully by waiting for the processing of all
  //received data to be completed

  ssc.getState()
```

Wonderful explanation on maintaining State using Pair DStreams:

http://asyncified.io/2016/07/31/exploring-stateful-streaming-with-apache-spark/

