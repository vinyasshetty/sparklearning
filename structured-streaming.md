Now we will be learning Spark Structured Streaming where we will be using DataSet/DataFrame API for Streaming related transformation.

```
//We will create a SparkSession object as usual
val spark = SparkSession().builder.appName("test1").master("local[*]").getOrCreate()
//We will have a 
val stream1 = spark.readStream //DataStreamReader
                   .format("socket") //This can text,json,csv,jdbc,etc
                   .option("hostname","localhost")
                   .option("port",5401)
                   .load()

val words = stream1.as[String].flatMap(_.split("\\s+"))
val cnts = words.groupBy($"value").count() //Is this a action???

val res = cnts.writeStream // DataStreamWriter
              .format("console")
              .outputMode("complete")
              .start() // DataStreamWriter has a start method which will start the computation
              //Thing to note here is there is NO save method unlike DataFrameWriter,so to save the 
              //result on hdfs,use .option("path","<location>")

res.awaitTermination()
```

In Structured Streaming,we think of a input stream as a unbounded table ,where as a data comes it becomes appended as new rows to the unbounded table.

