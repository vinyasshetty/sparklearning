Here we will talk about window and late events :

Now usually in streaming applications we will have timestamp embedded in the data ie : Say we have a toll company which has toll booths in multiple locations ,every time a vehicle passes ,through some detector information about the vehicle gets logged along with the timestamp when the vehicle passed .Now as a part of your streaming application you may want to do some calculation on the data for the last one hour or 30 mins ,then we would have to collect that information based on the timestamp which is part of the data.This timestamp is called the "Event Time". There is a different time called Processing time which is the time at which the data reached spark streaming.Though we say this is real time application,there will be some delay between the "event time" and "processing time" value and most of the time we would want to do calculation based on the "event time" and not processing time.

Now say a data looks like this :

```
vinyas,1,2018-03-17 09:04:21
namratha,2,2018-03-17 09:04:23
varsha,3,2018-03-17 09:04:33
vikas,4,2018-03-17 09:04:44
vidya,5,2018-03-17 09:04:25
```

Now i want to look at the count of records  for 10 seconds worth of the "Event Time" data.

For this spark provides a inbuilt "window function"

```
window($"ts","10 second")
```

Code would like this:

```
  val df1 = spark.readStream.format("socket").option("host","localhost").option("port","5432").load()

  val df2 = df1.as[String].map(x=>x.split(","))

  val df3 = df2.select($"value"(0).as("name"),$"value"(1).cast(IntegerType).as("id"),$"value"(2).cast(TimestampType).as("ts"))

  val df4 = df3.select($"*",window($"ts","10 second"))

  val df5 = df4.writeStream.trigger(Trigger.ProcessingTime(5 seconds)).option("truncate","false")
    .outputMode("update").format("console").start()

  df5.awaitTermination()
```

Now when you do a :

```
/*think about this as a new column called window is created and each record will have the "window value"
populated based on the ts column value of the current data.This is same as when you say do a groupBy($"id" % 2)
*/

If i give input as for above code:
Vinyass-MacBook-Pro:~ vinyasshetty$ nc -lk 5432
vinyas,1,2018-03-17 09:04:21
namratha,2,2018-03-17 09:04:23
varsha,3,2018-03-17 09:04:33
vikas,4,2018-03-17 09:04:44
vidya,5,2018-03-17 09:04:25

Spark will give result as,look at the new column window,spark decides on the starting range(ie 2018-03-17 09:04:20) 
based on the lowest value of ts in the current data :
-------------------------------------------
Batch: 0
-------------------------------------------
+--------+---+-------------------+------------------------------------------+
|name    |id |ts                 |window                                    |
+--------+---+-------------------+------------------------------------------+
|vinyas  |1  |2018-03-17 09:04:21|[2018-03-17 09:04:20, 2018-03-17 09:04:30]|
|namratha|2  |2018-03-17 09:04:23|[2018-03-17 09:04:20, 2018-03-17 09:04:30]|
|varsha  |3  |2018-03-17 09:04:33|[2018-03-17 09:04:30, 2018-03-17 09:04:40]|
|vikas   |4  |2018-03-17 09:04:44|[2018-03-17 09:04:40, 2018-03-17 09:04:50]|
|vidya   |5  |2018-03-17 09:04:25|[2018-03-17 09:04:20, 2018-03-17 09:04:30]|
+--------+---+-------------------+------------------------------------------+
```

** window function works only on timestamp datatype **

Now coming back to groupBy and count logic\(EventTimeSocket\) :

```
val df1 = spark.readStream.format("socket").option("host","localhost").option("port","5432").load()

  val df2 = df1.as[String].map(x=>x.split(","))

  val df3 = df2.select($"value"(0).as("name"),$"value"(1).cast(IntegerType).as("id"),$"value"(2).cast(TimestampType).as("ts"))

  val df4 = df3.groupBy(window($"ts","10 second")).count()

  val df5 = df4.writeStream.trigger(Trigger.ProcessingTime(5 seconds)).option("truncate","false")
    .outputMode("update").format("console").start()

  df5.awaitTermination()
```

Now this should be easy to understand and it will behave like any other regular group by .Just to iterate again the Trigger.ProcessingTime\(5 seconds\) ,this only means that spark will trigger the job to run every 5 seconds if there is any new data on the source side \(ie Only if the existing running job is completed\).This is same as your batch time in dstreams.

```
For above code If i pass input as :

Vinyass-MacBook-Pro:~ vinyasshetty$ nc -lk 5432
vinyas,1,2018-03-17 09:04:21
namratha,2,2018-03-17 09:04:23
varsha,3,2018-03-17 09:04:33

SPark O/p:
-------------------------------------------
Batch: 0
-------------------------------------------
+------------------------------------------+-----+
|window                                    |count|
+------------------------------------------+-----+
|[2018-03-17 09:04:20, 2018-03-17 09:04:30]|2    |
|[2018-03-17 09:04:30, 2018-03-17 09:04:40]|1    |
+------------------------------------------+-----+

Now in input i pass this :
vikas,4,2018-03-17 09:04:44
vidya,5,2018-03-17 09:04:25 <Enter>

Spark O/p:
-------------------------------------------
Batch: 1
-------------------------------------------
+------------------------------------------+-----+
|window                                    |count|
+------------------------------------------+-----+
|[2018-03-17 09:04:20, 2018-03-17 09:04:30]|3    |
|[2018-03-17 09:04:40, 2018-03-17 09:04:50]|1    |
+------------------------------------------+-----+

Now as you see it behaves as earlier.I have used update ouput mode.All the condition we spoke about earlier
holds good for append and complete.
```

Now as you saw above ,window is formed for 10 seconds worth of "event time" data.,but wat if we want to 10 seconds worth of "event time" data ,updated every 5 seconds in terms of "event data" \(NOT SAME AS Processing time\)

**EventTimeSocketSlide**

```
val df1 = spark.readStream.format("socket").option("host","localhost").option("port","5432").load()

  val df2 = df1.as[String].map(x=>x.split(","))

  val df3 = df2.select($"value"(0).as("name"),$"value"(1).cast(IntegerType).as("id"),$"value"(2).cast(TimestampType).as("ts"))

  val df4 = df3.select($"*",window($"ts","10 second","5 second")) //See the extra parameter

  val df5 = df4.writeStream.outputMode("update").option("truncate","false")
    .trigger(Trigger.ProcessingTime(10 seconds)).format("console").start()

  df5.awaitTermination()
```

```
For input :
Vinyass-MacBook-Pro:~ vinyasshetty$ nc -lk 5432
vinyas,1,2018-03-17 09:04:21
namratha,2,2018-03-17 09:04:23
varsha,3,2018-03-17 09:04:33
vikas,4,2018-03-17 09:04:44
vidya,5,2018-03-17 09:04:25

Spark O/p:
-------------------------------------------
Batch: 0
-------------------------------------------
+--------+---+-------------------+------------------------------------------+
|name    |id |ts                 |window                                    |
+--------+---+-------------------+------------------------------------------+
|vinyas  |1  |2018-03-17 09:04:21|[2018-03-17 09:04:15, 2018-03-17 09:04:25]|
|vinyas  |1  |2018-03-17 09:04:21|[2018-03-17 09:04:20, 2018-03-17 09:04:30]|
|namratha|2  |2018-03-17 09:04:23|[2018-03-17 09:04:15, 2018-03-17 09:04:25]|
|namratha|2  |2018-03-17 09:04:23|[2018-03-17 09:04:20, 2018-03-17 09:04:30]|
|varsha  |3  |2018-03-17 09:04:33|[2018-03-17 09:04:25, 2018-03-17 09:04:35]|
|varsha  |3  |2018-03-17 09:04:33|[2018-03-17 09:04:30, 2018-03-17 09:04:40]|
|vikas   |4  |2018-03-17 09:04:44|[2018-03-17 09:04:35, 2018-03-17 09:04:45]|
|vikas   |4  |2018-03-17 09:04:44|[2018-03-17 09:04:40, 2018-03-17 09:04:50]|
|vidya   |5  |2018-03-17 09:04:25|[2018-03-17 09:04:20, 2018-03-17 09:04:30]|
|vidya   |5  |2018-03-17 09:04:25|[2018-03-17 09:04:25, 2018-03-17 09:04:35]|
+--------+---+-------------------+------------------------------------------+

Look closely at window column ,range is still 10 seconds,but its every 5 seconds,this has a 
side effect that one record may be available in multiple grouping.Like vinyas is available in two ranges.
Hence number of records in the output will increase.
```

If we need to group by and count like earlier:

```
val df1 = spark.readStream.format("socket").option("host","localhost").option("port","5432").load()

val df2 = df1.as[String].map(x=>x.split(","))

val df3 = df2.select($"value"(0).as("name"),$"value"(1).cast(IntegerType).as("id"),$"value"(2).cast(TimestampType).as("ts"))

val df4 = df3.groupBy(window($"ts","10 second","5 second")).count()

val df5 = df4.writeStream.outputMode("update").option("truncate","false")
    .trigger(Trigger.ProcessingTime(10 seconds)).format("console").start()
```

```
For above code If i pass input as :

Vinyass-MacBook-Pro:~ vinyasshetty$ nc -lk 5432
vinyas,1,2018-03-17 09:04:21
namratha,2,2018-03-17 09:04:23
varsha,3,2018-03-17 09:04:33

-------------------------------------------
Batch: 0
-------------------------------------------
+------------------------------------------+-----+
|window                                    |count|
+------------------------------------------+-----+
|[2018-03-17 09:04:20, 2018-03-17 09:04:30]|2    |
|[2018-03-17 09:04:25, 2018-03-17 09:04:35]|1    |
|[2018-03-17 09:04:30, 2018-03-17 09:04:40]|1    |
|[2018-03-17 09:04:15, 2018-03-17 09:04:25]|2    |
+------------------------------------------+-----+

Now see due to the side effect ,we poske about sum of the count(2+1+1+2) is more the actual record count 
due to one record being available in differnt group range

Now when i pass :
vikas,4,2018-03-17 09:04:44
vidya,5,2018-03-17 09:04:25

-------------------------------------------
Batch: 1
-------------------------------------------
+------------------------------------------+-----+
|window                                    |count|
+------------------------------------------+-----+
|[2018-03-17 09:04:20, 2018-03-17 09:04:30]|3    |
|[2018-03-17 09:04:25, 2018-03-17 09:04:35]|2    |
|[2018-03-17 09:04:35, 2018-03-17 09:04:45]|1    |
|[2018-03-17 09:04:40, 2018-03-17 09:04:50]|1    |
+------------------------------------------+-----+
```

** While determining where a ts value fits in the group,Starting value of the range is inclusive but the ending value is excluded**. ie look at the vidya comes in 25 second ,so its included in range 25 to 35.But if we had a value as 35 second,then it would NOT be included in 25 to 35 range,but would come in the range of 30 to 40 and 35 to 45.

