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

Now DataSet will have all the methods that was available on dataframe but we will certain minor changes:

When you do operation like select,filter,groupBy,join etc on DataSet, it will return a DataFrame.To convert that back to DataSet you need again use a encoder

```
scala> ds1.select($"name",$"age")
res40: org.apache.spark.sql.DataFrame = [name: string, age: int]

scala> case class Emp(name:String,age:Int)
defined class Emp

scala> ds1.select($"name",$"age").as[Emp]
res41: org.apache.spark.sql.Dataset[Emp] = [name: string, age: int]
```



The Encoder object name and type should match with dataframe columns ,but you can have fewer and order also can be different.We will talk about map further.See the ordering and number of columns is different.

```
scala> case class Emp1(id:Int,name:String)
defined class Emp1

scala> ds1.select($"name",$"age",$"id").as[Emp1]
res49: org.apache.spark.sql.Dataset[Emp1] = [name: string, age: int ... 1 more field]

scala> ds1.select($"name",$"age",$"id").as[Emp1].map(x=>x.id).show
+-----+
|value|
+-----+
|    1|
|    2|
|    3|
+-----+


scala> ds1.select($"name",$"age",$"id").as[Emp1].map(x=>x.name).show
+--------+
|   value|
+--------+
|  vinyas|
|  shetty|
|namratha|
+--------+


scala> ds1.select($"name",$"age",$"id").as[Emp1].map(x=>x.age).show
<console>:37: error: value age is not a member of Emp1
       ds1.select($"name",$"age",$"id").as[Emp1].map(x=>x.age).show
                                                          ^

```



