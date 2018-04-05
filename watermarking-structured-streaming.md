Now talking about watermarking is little tricky.The actual use case of watermarking is explained well enuf in the SPark Streaming Guide,below is few tricky things i had to get my mind around.

Watermark has effect only on update and append mode.

Now as we saw earlier\(Window Chapter\) we are grouping records based on the window in which the "event time"  of the given records fall in.But in the output mode "update" ,we will keep information about all the earlier records also,sometimes we may not need that and we may want to retain information only upto certain time and this can be done with watermark.Watermark may not seem useful in terms of update ,but in append it really shines.

Till now i have told you that "append" output mode cannot have groupBy on the dataframes, but if we use watermark first and then use a groupBy operations on the same watermark column, spark allows this.Because now spark has a timeframe  till where it needs to keep track of previous result**.This is where watermark really shines the most.**

Let's talk more about both the above points :

Lets first start with "append" ouput mode with watermark.

**EventTimeWaterMark1\_Append**

```
 val df1 = spark.readStream.format("socket").option("host","localhost").option("port","5432").load()

  val df2 = df1.as[String].map(x=>x.split(","))

  val df3 = df2.select($"value"(0).as("name"),$"value"(1).cast(IntegerType).as("id"),$"value"(2).cast(TimestampType).as("ts"))

  val df4 = df3.withWatermark("ts","20 second").groupBy(window($"ts","10 second")).count()
   *** Water mark set to 20 second duration ***
   
  val df5 = df4.writeStream.outputMode("append").trigger(Trigger.ProcessingTime(8 seconds))
    .format("console").option("truncate","false").start()

  df5.awaitTermination()
```

Now as you see i have used withWatermark function and have given the ts column and the "20 second" as the amount of time i want spark to keep track of the  result data.Also as you see my output mode is "append" and i have still used groupBy function.This is the power of watermark on append.

```
**In the Notes below ,instead of saying 2018-03-17 09:04:33, for brevity sake i will just say 39 second**

I send this data as input
Vinyass-MacBook-Pro:~ vinyasshetty$ nc -lk 5432
vinyas,1,2018-03-17 09:04:21
namratha,2,2018-03-17 09:04:23
varsha,3,2018-03-17 09:04:33

Get Ready for suprise !!!,We get below
-------------------------------------------
Batch: 0
-------------------------------------------
+------+-----+
|window|count|
+------+-----+
+------+-----+

Now spark has started collecting the data but it does not give the output yet,because spark does NOT
know if more data will coming which may fall in this group.Spark will output the result in append mode only 
when it knows it does NOT have to update it again and only way it can get that gurantee is when the given data
is outside of the calculated watermark bound ie in the above data as you see,the maximum time is 33 second
,and we have given watermark as 20 second ,so spark will continue to track the result of data
which can (33-20 =  13 second) ie in the range of 10 to 20 and after and since we all the data is after 10 second,
spark will not output any result
Lets see this with example :

Next i Pass data :
srini,8,2018-03-17 09:04:59
pami,7,2018-03-17 09:04:27

Spark still gives output as :
-------------------------------------------
Batch: 1
-------------------------------------------
+------+-----+
|window|count|
+------+-----+
+------+-----+

So as per above the maximum time is 59 second in the data,so 59 - 20(watermark) == 39 ,so spark will understand 
that it does NOT have to keep track of any data which falls before the 30 to 40 range(since 39 falls in the 30 to 40 
range,spark keeps track of records greater then equal to 30 to 40 range) and spark is now ready to print the
result(Those in the range below 30) ,but it will do it only when the next trigger comes,is when next new data arrives.

Also one more point to note is that pami which as time as 27 actually in logical terminolgy has arrived late.
Ideally it should have come after namratha,but spark still took care of it ,this is due to watermark.This is
WHOLE GOAL of watermark that it keeps track of old data.

Next i Pass data:
julie,8,2018-03-17 09:04:44

Spark Output:
-------------------------------------------
Batch: 2
-------------------------------------------
+------------------------------------------+-----+
|window                                    |count|
+------------------------------------------+-----+
|[2018-03-17 09:04:20, 2018-03-17 09:04:30]|3    |
+------------------------------------------+-----+

Now if i Pass data as :
baalu,10,2018-03-17 09:04:23

-------------------------------------------
Batch: 3
-------------------------------------------
+------+-----+
|window|count|
+------+-----+
+------+-----+

Maximum time is still 59(Below POINT 3 IN Points to Rmber) and the tracking is supposed is of range(30 -39)

Next input data:
juil,11,2018-03-17 09:04:33
ram,12,2018-03-17 09:05:12

-------------------------------------------
Batch: 4
-------------------------------------------
+------+-----+
|window|count|
+------+-----+
+------+-----+

Now ram 5 minute 12 second becomes the maximun ts time ,so minus 20 second watermark will give as 4 minute 52 
second,so effectively means any data before 4 min 40sec - 50 sec range need NOT be tracked
and next time,we get any data ,it will be printed:

Next i pass :
buji,13,2018-03-17 09:05:15

As Expected:
-------------------------------------------
Batch: 5
-------------------------------------------
+------------------------------------------+-----+
|window                                    |count|
+------------------------------------------+-----+
|[2018-03-17 09:04:30, 2018-03-17 09:04:40]|2    |
|[2018-03-17 09:04:40, 2018-03-17 09:04:50]|1    |
+------------------------------------------+-----+
```

Points to remember:

1. withWatermark must be done first before any grouping is done.
2. Column on which you do the watermark should be the same column on which you do the group by and they should be timestamp type.
3. The maximum time which i did above is done on the entire dataset and not  on current record.
4. When we do Lower Bound = maximum time - \(watermark duration we have given\). Now any ranges and above in which this Lower bound is part of ,then those range values are still kept track of by spark\(which effectively means in append mode,they are NOT printed/outputed yet\).This Point becomes important,when we have slide also when we do window .

Update Mode\(EventTimeWaterMark1\_Update\) . We already know that update maintains history of earlier results/ouput,why we need this watermark feature, now the only reason i can think watermark is useful with update is since streaming applications runs for very long duration ,you may NOT want to keep track of all the history result,so you can set a limit.But having said that spark only gurantees that data coming within the lower bound is NOT LOST ,but does NOT gurantee that it will be filtered out.

Example to explain :

```
val df1 = spark.readStream.format("socket").option("host","localhost").option("port","5432").load()

val df2 = df1.as[String].map(x=>x.split(","))

val df3 = df2.select($"value"(0).as("name"),$"value"(1).cast(IntegerType).as("id"),$"value"(2).cast(TimestampType).as("ts"))

val df4 = df3.withWatermark("ts","10 second").groupBy(window($"ts","10 second")).count()
//As you see the watermark syntax has been used here.

val df5 = df4.writeStream.outputMode("update") //.trigger(Trigger.ProcessingTime(10 seconds))
    .format("console").option("truncate","false").start()

df5.awaitTermination()
```

```
******Have MADE THE WATERMARK 10 SECOND********
Input to Above :
Vinyass-MacBook-Pro:~ vinyasshetty$ nc -lk 5432
vinyas,1,2018-03-17 09:04:21
namratha,2,2018-03-17 09:04:23
varsha,3,2018-03-17 09:04:33

Spark OutPut :
-------------------------------------------
Batch: 0
-------------------------------------------
+------------------------------------------+-----+
|window                                    |count|
+------------------------------------------+-----+
|[2018-03-17 09:04:20, 2018-03-17 09:04:30]|2    |
|[2018-03-17 09:04:30, 2018-03-17 09:04:40]|1    |
+------------------------------------------+-----+

I pass this data:
vikas,4,2018-03-17 09:04:44
vidya,5,2018-03-17 09:04:25

-------------------------------------------
Batch: 1
-------------------------------------------
+------------------------------------------+-----+
|window                                    |count|
+------------------------------------------+-----+
|[2018-03-17 09:04:20, 2018-03-17 09:04:30]|3    |
|[2018-03-17 09:04:40, 2018-03-17 09:04:50]|1    |
+------------------------------------------+-----+


Now spark is behaving same way as it would have without "watermark" also.Now the maximum time is 
44 second(vikas) ,and watermark was 10 second,so the lower bound is 34,so spark need to keep track of 
30 to 40 range and above only and does NOT gurantee that it will keep track of anything before 30.

I pass data as :
julie,6,2018-03-17 09:04:24

Spark prints NOthing,this is where watermark makes update behave differently(but this different behaviour is
NOT guranteed),it could have very well behaved as if  there was NO watermark also and printed 20 to 30 range as 4 count
-------------------------------------------
Batch: 3
-------------------------------------------
+------+-----+
|window|count|
+------+-----+
+------+-----+



Now to put in simple terms ,in "update" mode ,spark works the same with or without watermarking ,provided
the data is within the lower bound(same way we calculated earlier).
Now if data is outside lower bound range,then when we have update with watermark ,
spark will NOT guarantee if it will behave the same way when we have only update.

Also unlike append,the update mode will give result then and there and it will NOT WAIT.
```

Now when we use watermark ,then we can group by without window also ,ie we directly group by  with watermarked column :

```
val df1 = spark.readStream.format("socket").option("host","localhost").option("port","5432").load()

  val df2 = df1.as[String].map(x=>x.split(","))

  val df3 = df2.select($"value"(0).as("name"),$"value"(1).cast(IntegerType).as("id"),$"value"(2).cast(TimestampType).as("ts"))

  val df4 = df3.withWatermark("ts","20 second").groupBy($"ts").count()

  val df5 = df4.writeStream.outputMode("append") //.trigger(Trigger.ProcessingTime(10 seconds))
    .format("console").option("truncate","false").start()
```

**So watermark provides append with group by capability ,but only with timestamp column datatype**

**Also watermark has NO effect on "complete" ouput mode ,it  just ignores it.**

