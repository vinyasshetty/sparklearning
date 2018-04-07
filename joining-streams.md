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

**Inner Join without any watermarking** ** StreamtToStream1 **:

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

Spark docs has clearly told that in Stream-Stream join ,they maintain the state of both the streams:

```
The challenge of generating join results between two data streams is that, at any point of time, 
the view of the dataset is incomplete for both sides of the join making it much harder to find matches 
between inputs. Any row received from one input stream can match with any future, yet-to-be-received 
row from the other input stream. Hence, for both the input streams, we buffer past input as streaming state, 
so that we can match every future input with past input and accordingly generate joined results.
```

Lets see some examples to run above code :

```
Lets start with simple ones:

Vinyass-MacBook-Pro:~ vinyasshetty$ nc -lk 5431
vinyas,1,2018-03-17 09:04:21
namratha,2,2018-03-17 09:04:23
varsha,3,2018-03-17 09:04:33

Vinyass-MacBook-Pro:~ vinyasshetty$ nc -lk 5430
vinyas,1,2018-03-17 09:04:21
namratha,2,2018-03-17 09:04:23
varsha,3,2018-03-17 09:04:33

Now same set of keys are avialable in the data and when Trigger Processing time comes,
we get spark output as :
-------------------------------------------
Batch: 0
-------------------------------------------
+--------+---+-------------------+--------+---+-------------------+
|name    |id |ts                 |name    |id |ts                 |
+--------+---+-------------------+--------+---+-------------------+
|vinyas  |1  |2018-03-17 09:04:21|vinyas  |1  |2018-03-17 09:04:21|
|varsha  |3  |2018-03-17 09:04:33|varsha  |3  |2018-03-17 09:04:33|
|namratha|2  |2018-03-17 09:04:23|namratha|2  |2018-03-17 09:04:23|
+--------+---+-------------------+--------+---+-------------------+

This is straight forward.
```

```
Now with a new fresh run ,lets do this :

Data has come on one stream :
Vinyass-MacBook-Pro:~ vinyasshetty$ nc -lk 5431
vinyas,1,2018-03-17 09:04:21
namratha,2,2018-03-17 09:04:23

But nothing yet on the other:
Vinyass-MacBook-Pro:~ vinyasshetty$ nc -lk 5430

Now spark sees,data has changed and when trigger time recahes,it gives output as :
-------------------------------------------
Batch: 0
-------------------------------------------
+----+---+---+----+---+---+
|name|id |ts |name|id |ts |
+----+---+---+----+---+---+
+----+---+---+----+---+---+

No data ,since its a inner join.

Next ,the second stream gets data :
Vinyass-MacBook-Pro:~ vinyasshetty$ nc -lk 5430
vinyas,1,2018-03-17 09:04:21

Spark ouput:,as told earlier,spark is keep the state and when second stream comes,it will join stream1 data(state
which it had kept) with new stream2 data.
-------------------------------------------
Batch: 1
-------------------------------------------
+------+---+-------------------+------+---+-------------------+
|name  |id |ts                 |name  |id |ts                 |
+------+---+-------------------+------+---+-------------------+
|vinyas|1  |2018-03-17 09:04:21|vinyas|1  |2018-03-17 09:04:21|
+------+---+-------------------+------+---+-------------------+


Now lets say ,we again get fresh set of data on both streams:
5431:
rini,8,2018-03-17 09:04:59
pami,7,2018-03-17 09:04:27
juil,11,2018-03-17 09:04:33

5430:
rini,8,2018-03-17 09:04:59
pami,7,2018-03-17 09:04:27

Spark output(no surprises):
-------------------------------------------
Batch: 3
-------------------------------------------
+----+---+-------------------+----+---+-------------------+
|name|id |ts                 |name|id |ts                 |
+----+---+-------------------+----+---+-------------------+
|rini|8  |2018-03-17 09:04:59|rini|8  |2018-03-17 09:04:59|
|pami|7  |2018-03-17 09:04:27|pami|7  |2018-03-17 09:04:27|
+----+---+-------------------+----+---+-------------------+


Now both 5431 and 5430 had received id 1 ,so lets send id 1 again in 5430 only
vinyas,1,2018-03-17 09:04:28 

Spark output:
-------------------------------------------
Batch: 4
-------------------------------------------
+------+---+-------------------+------+---+-------------------+
|name  |id |ts                 |name  |id |ts                 |
+------+---+-------------------+------+---+-------------------+
|vinyas|1  |2018-03-17 09:04:21|vinyas|1  |2018-03-17 09:04:28|
+------+---+-------------------+------+---+-------------------+

**Now somehow sparks knows to take only the latest id 1 data from 5430 and join that with old 5431 data.
it ignores the old 5430 data.*****
```

```
As per my understanding,they way it works is :
1)Past States/data of both streams are maintained.
2)Now whenever trigger happens,along with a change in one of the streams,
then first(think would be left one or the one which has changed first) streams latest records is joined 
with latest and old records of stream2
3)Now next latest stream2 records which have not been joined already in step2 will check if they can be joined with 
old stream1 records.
So point 2 and Point 3 may cause some duplicate scenarios
```

**Inner Join with watermarking** ** StreamToStream1\_1 **:

```
val df1 = spark.readStream.format("socket").option("host","localhost").option("port","5431").load()

  val df2 = df1.as[String].map(x=>x.split(","))

  val df3 = df2.select($"value"(0).as("name"),
    $"value"(1).cast(IntegerType).as("id"),
    $"value"(2).cast(TimestampType).as("ts")).withWatermark("ts","15 second")

  val df1_1 = spark.readStream.format("socket").option("host","localhost").option("port","5430").load()

  val df2_1 = df1_1.as[String].map(x=>x.split(","))

  val df3_1 = df2_1.select($"value"(0).as("name"),$"value"(1).cast(IntegerType).as("id"),
    $"value"(2).cast(TimestampType).as("ts")).withWatermark("ts","10 second")

  val joindf = df3.join(df3_1,df3("id") <=> df3_1("id"))

  val res = joindf.writeStream.outputMode("append").trigger(Trigger.ProcessingTime(15 seconds))
    .format("console").option("truncate","false").start()

Now with this,watermark value is set,ie before the join happens:
For every stream,We check whats the highest value ts(till date) and based on that we form the lower bound,
if the records have ts within the lower bound,then they are considered for the joins.
If join is possible,then join happens then and there and it does NOT wait for the records to becomes 
out of lower bound to populate the result.
```

```
Vinyass-MacBook-Pro:~ vinyasshetty$ nc -lk 5431
vinyas,1,2018-03-17 09:04:21
namratha,2,2018-03-17 09:04:23


Vinyass-MacBook-Pro:~ vinyasshetty$ nc -lk 5430
vinyas,1,2018-03-17 09:04:21
namratha,2,2018-03-17 09:04:23

-------------------------------------------
Batch: 0
-------------------------------------------
+--------+---+-------------------+--------+---+-------------------+
|name    |id |ts                 |name    |id |ts                 |
+--------+---+-------------------+--------+---+-------------------+
|vinyas  |1  |2018-03-17 09:04:21|vinyas  |1  |2018-03-17 09:04:21|
|namratha|2  |2018-03-17 09:04:23|namratha|2  |2018-03-17 09:04:23|
+--------+---+-------------------+--------+---+-------------------+

In 5431 and 5430 i pass:
rini,8,2018-03-17 09:04:53

-------------------------------------------
Batch: 1
-------------------------------------------
+----+---+-------------------+----+---+-------------------+
|name|id |ts                 |name|id |ts                 |
+----+---+-------------------+----+---+-------------------+
|rini|8  |2018-03-17 09:04:59|rini|8  |2018-03-17 09:04:59|
+----+---+-------------------+----+---+-------------------+

Now the highest ts is 59 second,so lower bound for stream1 -> 59-15 = 44 and stream2 -> 59-10 = 49

I pass in 541 and 5430 :
pami,7,2018-03-17 09:04:27

Since 27 is out of range in both streams,its NOT considered
-------------------------------------------
Batch: 2
-------------------------------------------
+----+---+---+----+---+---+
|name|id |ts |name|id |ts |
+----+---+---+----+---+---+
+----+---+---+----+---+---+


Only 5431 :
rini,8,2018-03-17 09:04:53
Spark output
-------------------------------------------
Batch: 3
-------------------------------------------
+----+---+-------------------+----+---+-------------------+
|name|id |ts                 |name|id |ts                 |
+----+---+-------------------+----+---+-------------------+
|rini|8  |2018-03-17 09:04:53|rini|8  |2018-03-17 09:04:59|
+----+---+-------------------+----+---+-------------------+


Now 5431 stream1 lowerbound is 44 and stream2(5430) lowerbound is 49,so i pass :
5431(Considered out of scope) :
vicky,11,2018-03-17 09:04:42

5430(In scope):
vicky,11,2018-03-17 09:04:51

-------------------------------------------
Batch: 4
-------------------------------------------
+----+---+---+----+---+---+
|name|id |ts |name|id |ts |
+----+---+---+----+---+---+
+----+---+---+----+---+---+


Now in 5431 ,i pass:
vicky,11,2018-03-17 09:04:45

-------------------------------------------
Batch: 7
-------------------------------------------
+-----+---+-------------------+-----+---+-------------------+
|name |id |ts                 |name |id |ts                 |
+-----+---+-------------------+-----+---+-------------------+
|vicky|11 |2018-03-17 09:04:45|vicky|11 |2018-03-17 09:04:51|
+-----+---+-------------------+-----+---+-------------------+

Now ,as the watermarking guarantee as earlier,watermarking guarantee is only ONE SIDE.
ie it make sure records with a lower bound will NOT be lost,but it DOES NOT gurantee that records 
outside the lower bound will always be LOST.
```

With Inner Stream-Stream1 Join,watermarking was NOT manadtory,but with left/right outer join its mandatory because as spark docs says:

```
While the watermark + event-time constraints is optional for inner joins,
for left and right outer joins they must be specified. This is because for 
generating the NULL results in outer join, the engine must know when an input 
row is not going to match with anything in future. Hence, the watermark + event-time 
constraints must be specified for generating correct results.
```

Now as we know spark stream-stream join supports only append mode and a feature in append mode is once it output the result,it cannot update it.SO as we saw in aggregation append watermark ,it would give the result only once the records went outside the lower bound and there was NO need to update them,same thing happens in "left/right" outer joins ,because before spark outputs a null and says records where NOT available,it has to make sure it will NOT come in future ,so that guarantee  is provided with watermark and hence like aggregation ,left/right outer join with output the result only once the records has ts that have become outside of lower bound. But in inner join ,we had NO such requirement,because if a record was there it would join else it would not join and give empty result.

Now along with watermark,we need one more filtering that needs to be done along with join for stream-stream left/right outer joins ** StreamtToStream2 **:

So if you use left outer join,we mandatorily need watermaking on right stream and join wise filtering on event time.left stream watermarking is optional.

Similarly for right outer join ,we mandatorily need watermaking on left stream and join wise filtering on event time.right stream watermarking is optional.

Below i have have used watermarking on both streams,but since this was left outer, watermarking on df3 is optional.

```
val df1 = spark.readStream.format("socket").option("host","localhost").option("port","5431").load()

val df2 = df1.as[String].map(x=>x.split(","))

val df3 = df2.select($"value"(0).as("name"),
    $"value"(1).cast(IntegerType).as("id"),
    $"value"(2).cast(TimestampType).as("ts")).withWatermark("ts","10 second")

val df1_1 = spark.readStream.format("socket").option("host","localhost").option("port","5430").load()

val df2_1 = df1_1.as[String].map(x=>x.split(","))

val df3_1 = df2_1.select($"value"(0).as("name"),
    $"value"(1).cast(IntegerType).as("id"),
    $"value"(2).cast(TimestampType).as("ts")).withWatermark("ts","10 second")

val jfilter = df3("ts") >= df3_1("ts") && df3("ts") <= df3_1("ts") + expr("INTERVAL 1 hour")
  **This is is extra FILTER **

val joindf = df3.join(df3_1,df3("id") <=> df3_1("id") && jfilter,"left_outer")



val res = joindf.writeStream.outputMode("append").trigger(Trigger.ProcessingTime(5 seconds))
    .format("console").option("truncate","false").start()
```

```

```



