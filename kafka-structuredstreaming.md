Here we talk about the integration about kafka and strcutured streaming.

Lets create a kafka topic and write some data to it.:

Kafka Producer:

```
object ProducerText extends App{

  val prop = new Properties()
  prop.setProperty("bootstrap.servers","127.0.0.1:9092,127.0.0.1:9093")
  prop.setProperty("key.serializer",classOf[IntegerSerializer].getName)
  prop.setProperty("value.serializer",classOf[StringSerializer].getName)
  prop.setProperty("acks","-1")
  prop.setProperty("retries","3")
  //prop.setProperty("transactional.id","cards-trans")

  //prop.setProperty("max.in.flight.requests.per.connection","1") //If i want to maintain order

  val producer = new KafkaProducer[Int,String](prop)


  val record1 = new ProducerRecord[Int,String]("cards",123,"123,vinyas,shetty,4598.32,2017-03-31,2018-02-25 08:02:23")
  val record2 = new ProducerRecord[Int,String]("cards",124,"124,namratha,rao,4562.51,2017-03-31,2018-02-25 08:02:25")
  val record3 = new ProducerRecord[Int,String]("cards",125,"125,vidya,shetty,3145.95,2017-03-31,2018-02-25 08:02:30")
  val record0 = new ProducerRecord[Int,String]("cards",120,"120,varsha,shetty,3145.95,2017-03-31,2018-02-25 08:02:22")
  val record4 = new ProducerRecord[Int,String]("cards",126,"126,abhishek,shetty,4612.87,2017-03-31,2018-02-25 08:02:42")
  val record5 = new ProducerRecord[Int,String]("cards",127,"127,shrinivas,shetty,5672.56,2017-03-31,2018-02-25 08:02:55")


  val recordList = record1::record2::record3::record0::record4::record5::Nil

  val latch = new CountDownLatch(6)

  try {
    for (x <- recordList) {
      producer.send(x, new Callback {
        override def onCompletion(metadata: RecordMetadata, exception: Exception): Unit = {
          if (exception == null) {
            println(s"Successfully written for ${metadata.partition()} and ${metadata.toString}")
          }
          else {
            exception.printStackTrace()
          }
          latch.countDown()
        }
      })
    }

    latch.await()
  }
  finally{
    producer.close()
  }
}
```

Read more about Kafka : [https://vinyasshetty.gitbooks.io/kafka/content/](https://vinyasshetty.gitbooks.io/kafka/content/)

Now lets write  Spark Code \(KafkaWordCount\):

```
val spark = SparkSession.builder().appName("KafkaWc").master("local[*]").getOrCreate()

  import spark.implicits._
  spark.sparkContext.setLogLevel("Error")

  val df1 = spark.readStream
    .format("kafka")
    .option("kafka.bootstrap.servers","127.0.0.1:9092,127.0.0.1:9093")
    .option("subscribe","cards")
    .option("startingOffsets","earliest")
    .load()

  df1.printSchema()

  val df2 = df1.select($"key".cast("String"),$"value".cast("String")).as[(String,String)]

  val df3 = df2.map(x=>x._2).flatMap(x=>x.split(",")).groupBy($"value").count()

  val df4 = df3.writeStream.format("console")
    .trigger(Trigger.ProcessingTime(10 seconds))
    .option("truncate","false")
    .option("numRows","50")
    .outputMode("complete")
    .start()

  df4.awaitTermination()
```

Now When spark reads from Kafka ,the dataframe it gets has the below schema\(df1\)  ie 7 columns

```
root
 |-- key: binary (nullable = true)
 |-- value: binary (nullable = true)
 |-- topic: string (nullable = true)
 |-- partition: integer (nullable = true)
 |-- offset: long (nullable = true)
 |-- timestamp: timestamp (nullable = true)
 |-- timestampType: integer (nullable = true)

 As you see key and value comes in binary format and they can be converted first to String only and 
 then if need be you can cast to other types.
```

** Observed that key was being pouplated as null even after i casted to string in spark,but in Kafka producer if i changed the key type to string and write it then spark was able to convert them correctly when  i casted them to string.**

```
//This is after i changed above KafkaProducer Code to have key as String and wrote to a 
new topic called cards1
-------------------------------------------
Batch: 0
-------------------------------------------
+---+-----------------------------------------------------------+------+---------+------+-----------------------+-------------+
|key|value                                                      |topic |partition|offset|timestamp              |timestampType|
+---+-----------------------------------------------------------+------+---------+------+-----------------------+-------------+
|125|125,vidya,shetty,3145.95,2017-03-31,2018-02-25 08:02:30    |cards1|0        |0     |2018-04-26 19:38:07.262|0            |
|123|123,vinyas,shetty,4598.32,2017-03-31,2018-02-25 08:02:23   |cards1|2        |0     |2018-04-26 19:38:07.254|0            |
|124|124,namratha,rao,4562.51,2017-03-31,2018-02-25 08:02:25    |cards1|2        |1     |2018-04-26 19:38:07.261|0            |
|126|126,abhishek,shetty,4612.87,2017-03-31,2018-02-25 08:02:42 |cards1|2        |2     |2018-04-26 19:38:07.262|0            |
|120|120,varsha,shetty,3145.95,2017-03-31,2018-02-25 08:02:22   |cards1|1        |0     |2018-04-26 19:38:07.262|0            |
|127|127,shrinivas,shetty,5672.56,2017-03-31,2018-02-25 08:02:55|cards1|1        |1     |2018-04-26 19:38:07.262|0            |
+---+-----------------------------------------------------------+------+---------+------+-----------------------+-------------+
```

## Offsets 

In Structured Streaming,we can read data from a particular partition and from a from a particular offset range.



