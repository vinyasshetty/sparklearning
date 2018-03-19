* DataSet is a distributed collection of typed data.
* It needs a encoder which will convert your scala types to Spark Internal types.Now i know DataFrame is just a DataSet\[Row\] ,but below i will talk as if they are two separate things just for my understanding to understand its subtle differences.In package.scala of sql =&gt; type DataFrame = Dataset\[Row\]
* Things to note below,when you read a external data ie SparkSession read method will still return a DataFrameReader,so you would have first create a dataframe and then use a Encoder \(with as\) and convert that to a Scala object\(below Employee object\)

```
bash-4.1$ hadoop fs -cat /user/lg489741/test1/vin.txt
id,name,age,amt
1,vinyas,28,76.34
2,shetty,30,67.2
3,namratha,28,65.2

scala> import org.apache.spark.sql.types._
import org.apache.spark.sql.types._

scala> val sch = StructType(StructField("id",IntegerType)::StructField("name",StringType)::StructField("age",IntegerType)::StructField("amt",DoubleType)::Nil)
sch: org.apache.spark.sql.types.StructType = StructType(StructField(id,IntegerType,true), StructField(name,StringType,true), StructField(age,IntegerType,true), StructField(amt,DoubleType,true))

scala> val df1 = spark.read.schema(sch).option("header",true).csv("hdfs://nn1/user/lg489741/test1/vin.txt")

scala> case class Employee(id:Int,name:String,age:Int,amt:Double)
defined class Employee


scala> val ds1 = df1.as[Employee]
ds1: org.apache.spark.sql.Dataset[Employee] = [id: int, name: string ... 2 more fields]
```

Now DataSet will have all the methods that was available on dataframe but we will certain minor changes:

When you do operation like** \*select, join ,agg,explode,withColumn,withColumnRenamed,drop,describe,summary** on DataSet, it will return a DataFrame. Seems like whenever there is a possibility of the Columns changing then such datasets return a Dataframe which would make sense since we don't know what new structure we would get.

To convert that back to DataSet you need again use a encoder

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

We can select on DataSet return a DataSet but this will have a encoder of Tuple.

```
scala> val ds99 = ds1.select($"name".as[String],$"age".as[Int])   //This can go only upto 5 columns.
ds99: org.apache.spark.sql.Dataset[(String, Int)] = [name: string, age: int]

//Can still do this
scala> ds1.select($"name".as[String],$"age".as[Int]).select($"name") 
res58: org.apache.spark.sql.DataFrame = [name: string]


//But not below since x is a tuple now
scala> ds99.map(x=>x.name)
<console>:37: error: value name is not a member of (String, Int)
       ds99.map(x=>x.name)
                     ^

scala> ds99.map(x=>x._1)
res61: org.apache.spark.sql.Dataset[String] = [value: string]

scala> ds99.map(x=>x._2)
res62: org.apache.spark.sql.Dataset[Int] = [value: int]
```

Special "joinWith" which returns a DataSet\[\(T,U\)\] .We can joinWith two DataSets of type T and U and it returns a DataSet\[\(T,U\)\]

```
scala> val j1 = ds1.joinWith(ds2,ds1("id") <=> ds2("id"))
j1: org.apache.spark.sql.Dataset[(Employee, Employee)] = [_1: struct<id: int, name: string ... 2 more fields>, _2: struct<id: int, name: string ... 2 more fields>]

//As you see it actually returns a DataSet of type Tuple2[T,U] where T is type of DataSet1
// which is joined with DataSet2 of type U.The columns names are _1 and _2.This is same behaviour as above.

scala> j1.schema
res66: org.apache.spark.sql.types.StructType = StructType(StructField(_1,StructType(StructField(id,IntegerType,true), StructField(name,StringType,true), StructField(age,IntegerType,true), StructField(amt,DoubleType,true)),false), StructField(_2,StructType(StructField(id,IntegerType,true), StructField(name,StringType,true), StructField(age,IntegerType,true), StructField(amt,DoubleType,true)),false))

scala> j1.columns
res67: Array[String] = Array(_1, _2)

scala> j1.map(x=>x)
res68: org.apache.spark.sql.Dataset[(Employee, Employee)] = [_1: struct<id: int, name: string ... 2 more fields>, _2: struct<id: int, name: string ... 2 more fields>]

scala> j1.map(x=>x._1)
res69: org.apache.spark.sql.Dataset[Employee] = [id: int, name: string ... 2 more fields]

scala> j1.select($"_1"("name"))
res73: org.apache.spark.sql.DataFrame = [_1.name: string]
```

"joinWith" with two dataframes .df1 and df2 are two dataframes

```
scala> val j2 = df1.joinWith(df2,df1("id") <=> df2("id"))
j2: org.apache.spark.sql.Dataset[(org.apache.spark.sql.Row, org.apache.spark.sql.Row)] = [_1: struct<id: int, name: string ... 2 more fields>, _2: struct<id: int, name: string ... 2 more fields>]

scala> j2.select($"_1"("name"))
res76: org.apache.spark.sql.DataFrame = [_1.name: string]

//Difference it its a DataSet[(Row,Row)]
```

We can use all the map , flatMap , mapPartitions operations on DataSet,but be careful on its return type.

```
//As you see it returns a DataSet[Int]
scala> ds1.map(r=>r.id+10)
res0: org.apache.spark.sql.Dataset[Int] = [value: int]

scala> ds1.map(r=>(r.name,r.id+10))
res1: org.apache.spark.sql.Dataset[(String, Int)] = [_1: string, _2: int]

scala> ds1.map(r=>(r.name,r.id+10)).select($"_1",$"_2").show
+--------+---+
|      _1| _2|
+--------+---+
|  vinyas| 11|
|  shetty| 12|
|namratha| 13|
+--------+---+


scala> ds1.map(r=>Emp99(r.id,r.amt+10))
res4: org.apache.spark.sql.Dataset[Emp99] = [id: int, amt: double]

scala> ds1.map(r=>Emp99(r.id,r.amt+10)).show
+---+-----+
| id|  amt|
+---+-----+
|  1|86.34|
|  2| 77.2|
|  3| 75.2|
+---+-----+


scala> ds1.flatMap(r=>List(Emp99(r.id,r.amt+10)))
res9: org.apache.spark.sql.Dataset[Emp99] = [id: int, amt: double]

scala> ds1.flatMap(r=>List(Emp99(r.id,r.amt+10))).show
+---+-----+
| id|  amt|
+---+-----+
|  1|86.34|
|  2| 77.2|
|  3| 75.2|
+---+-----+
```

We can persist/cache a DataSet also.

We have groupByKey operator on a DataSet.

```
ds.groupByKey(x=>x.name) // Now it will group by x.name,This returns 
a KeyValueGroupedDataset which is different from what is returned by groupBy ie RelationalGroupedDataset


import org.apache.spark.sql.types._

case class Employee(id:Int,name:String,amt:Double)
val sch = new StructType().add("id",IntegerType).add("name",StringType).add("amt",DoubleType) 
val ds1 = spark.read.schema(sch).csv("/FileStore/tables/vin.csv").as[Employee]
ds1.show

import org.apache.spark.sql.functions._
ds1.groupByKey(x=>x.id).agg(sum($"amt").as("summed").as[Double]).show
```

Aggregation :

```
scala> val ids = spark.range(10)
ids: org.apache.spark.sql.Dataset[Long] = [id: bigint]

scala> ids.agg(sum($"id").as("sum"))
res0: org.apache.spark.sql.DataFrame = [sum: bigint]

/*
Now as you see ,we can run all aggregations operator directly on the DataSet,since if you look the DataSet code,
agg operator first does a empty groupBy() ie groupBy on all the columns and then does a agg.

*/
```



