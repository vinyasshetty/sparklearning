* DataSet is a distributed collection of typed data.
* It needs a encoder which will convert your scala types to Spark Internal types.
* Things to note below,when you read a external data ie SparkSession read method will still return a DataFrameReader,so you would have first create a dataframe and then use a Encoder \(with as\) and convert that to a Scala object\(below Employee object\)

```
bash-4.1$ hadoop fs -cat /user/lg489741/test1/vin.txt
id,name,age,amt
1,vinyas,28,76.34
2,shetty,30,67.2
3,namratha,28,65.2

scala> val sch = StructType(StructField("id",IntegerType)::StructField("name",StringType)::StructField("age",IntegerType)::StructField("amt",DoubleType)::Nil)
sch: org.apache.spark.sql.types.StructType = StructType(StructField(id,IntegerType,true), StructField(name,StringType,true), StructField(age,IntegerType,true), StructField(amt,DoubleType,true))

scala> val df1 = spark.read.schema(sch).option("header",true).csv("hdfs://nn1/user/lg489741/test1/vin.txt")

scala> case class Employee(id:Int,name:String,age:Int,amt:Double)
defined class Employee


scala> val ds1 = df1.as[Employee]
ds1: org.apache.spark.sql.Dataset[Employee] = [id: int, name: string ... 2 more fields]

```



