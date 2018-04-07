DropDuplicates:

Spark keeps tracking of all the records and emits only one set of records,if the id had already come in the fast ,then it will not output it again.

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





