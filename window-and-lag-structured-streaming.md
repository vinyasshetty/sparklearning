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

Now i want to look at the count of records every 10 seconds of the "Event Time".

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

//think about this as a new column called window is created and each record will have the "window value"
//populated based on the ts column value .This is same as when you say do a groupBy($"id" % 2)

If i give input as :
Vinyass-MacBook-Pro:~ vinyasshetty$ nc -lk 5432
vinyas,1,2018-03-17 09:04:21
namratha,2,2018-03-17 09:04:23
varsha,3,2018-03-17 09:04:33
vikas,4,2018-03-17 09:04:44
vidya,5,2018-03-17 09:04:25

Spark will give result as,look at the new column window,spark decides on the starting range based on the lowest
value id the data :
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

Now coming back to groupBy and count logic :

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



