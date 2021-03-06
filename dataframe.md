## DataFrame:

Well DataFrame is nothing but

```
type DataFrame = Dataset[Row]
```

Dataframe transformation has predefined syntax rather than taking a function like a RDD transformation ,now this helps the catalyst to optimize the transformation.

Now Row\(1,2,3\) object actually returns a object of class

```
class GenericRow(protected[sql] val values: Array[Any]) extends Row

//We also a have trait by name Row which has most of the methods ,like getInt,get,fromSeq etc
```

```
scala> val df1 = spark.createDataFrame(List(Person(1,"vin"),Person(2,"she")) )
df1: org.apache.spark.sql.DataFrame = [id: int, name: string]

scala> val df1 = spark.createDataFrame(List(Person(1,"vin"),Person(2,"she")))
df1: org.apache.spark.sql.DataFrame = [id: int, name: string]

//DataFrame is made of Row object and henace a take on it will result a Array[Row]
scala> df1.take(2)
res7: Array[org.apache.spark.sql.Row] = Array([1,vin], [2,she])

scala> val ds1 = spark.createDataset(List(Person(1,"vin"),Person(2,"she")))
ds1: org.apache.spark.sql.Dataset[Person] = [id: int, name: string]

scala> ds1.take(2)
res8: Array[Person] = Array(Person(1,vin), Person(2,she))
```

Column is just a logical representation of every value .These have ability of to perform operation which we call as expressions.

Since Column is just a logical representation ,it is useful only when it is tied to dataframe which has the actual data in the form of Row objects.So we can define what operations we want to do on a column and then tie to a dataframe to actually do that operations on the data.

```
scala> df1
res10: org.apache.spark.sql.DataFrame = [id: int, name: string]

//Using implicits $"id" is converted to a Column("id")
scala> val cols1 = $"id" + 10
cols1: org.apache.spark.sql.Column = (id + 10)

scala> val col2 = $"name"
col2: org.apache.spark.sql.ColumnName = name

scala> df1.select(cols1,col2)
res11: org.apache.spark.sql.DataFrame = [(id + 10): int, name: string]

scala> df1.select(cols1,col2).show
+---------+----+
|(id + 10)|name|
+---------+----+
|       11| vin|
|       12| she|
+---------+----+
```

Different ways to access to column:

```
scala> df1.select($"id",df1.col("id"),df1("id"),'id,expr("id"),col("id"),column("id")).show
+---+---+---+---+---+---+---+
| id| id| id| id| id| id| id|
+---+---+---+---+---+---+---+
|  1|  1|  1|  1|  1|  1|  1|
|  2|  2|  2|  2|  2|  2|  2|
+---+---+---+---+---+---+---+

//Personally i Prefer the $ or the tick.But you may have to use df1.col or df1() ie associated with dataframe name
//when using join and have same column coming from two different dataframes.We will see this ahead.All of these
// types return a Column.

//Select has a option to take directly string:
scala> df1.select("id")
res24: org.apache.spark.sql.DataFrame = [id: int]

scala> df1.select("id").show
+---+
| id|
+---+
|  1|
|  2|
+---+

//Below will NOT work,ie you cant mix Column type and string.Look at the DataSet.scala select method.
scala> df1.select("id",$"id").show
```

If we want to create a column with a hardcoded value the use lit:

```
scala> df1.select($"*",lit(0.0).as("amt"))
res28: org.apache.spark.sql.DataFrame = [id: int, name: string ... 1 more field]

scala> df1.select($"*",lit(0.0).as("amt")).show
+---+----+---+
| id|name|amt|
+---+----+---+
|  1| vin|0.0|
|  2| she|0.0|
+---+----+---+


scala> df1.select($"*",lit(0.0).as("amt")).printSchema
root
 |-- id: integer (nullable = false)
 |-- name: string (nullable = true)
 |-- amt: double (nullable = false)

lit(<>) will return a Column with only one value.You can do all the operation you can do on a Column on a lit also.
```

Create a new Column :

```
scala> df1.withColumn("upperName",upper($"name")).show
+---+----+---------+
| id|name|upperName|
+---+----+---------+
|  1| vin|      VIN|
|  2| she|      SHE|
+---+----+---------+

//Renaming Column:
scala> df1.withColumnRenamed("name","name1").show
+---+-----+
| id|name1|
+---+-----+
|  1|  vin|
|  2|  she|
+---+-----+
```

Drop Columns:

```
scala> df1.drop($"name",$"id")
<console>:36: error: type mismatch;
 found   : org.apache.spark.sql.ColumnName
 required: String
       df1.drop($"name",$"id")
                ^
<console>:36: error: type mismatch;
 found   : org.apache.spark.sql.ColumnName
 required: String
       df1.drop($"name",$"id")
                        ^

scala> df1.drop($"name")
res39: org.apache.spark.sql.DataFrame = [id: int]

scala> df1.drop("name","id")
res40: org.apache.spark.sql.DataFrame = []

//PS : df1 will still be the same since dataframe is immutable ,it just returns a new dataframe with no column
```

Filtering:

```
scala> df1.filter($"name" === "vin" && $"id" === 1).show
+---+----+
| id|name|
+---+----+
|  1| vin|
+---+----+

Equality -> === (THIS IS NOT NULL SAFE)
Inequality -> =!=  (THIS IS NOT NULL)
Null safe equality -> <=> 

//Should have returned record
scala> df1.select(lit(null).as("ColNull")).filter($"ColNull" === null).show
+-------+
|ColNull|
+-------+
+-------+

//Should have returned records
scala> df1.select(lit(null).as("ColNull")).filter(!($"ColNull" =!= null)).show
+-------+
|ColNull|
+-------+
+-------+


//Null safe equality operation
scala> df1.select(lit(null).as("ColNull")).filter($"ColNull" <=> null).show
+-------+
|ColNull|
+-------+
|   null|
|   null|
+-------+

//NUll safe Not equality operation
scala> df1.select(lit(null).as("ColNull")).filter(!($"ColNull" <=> null)).show
+-------+
|ColNull|
+-------+
+-------+
```

DataFrameNaFunctions:

```
We can drop records with null or NaN values,we can fill the NaN or null value with values you have given and 
also we can replace values matching a given value.Will come to back
```

## Window Function:

Window function behaves a little like groupby but the main difference is it will give a result for every record in the input and not only for unique groups ie the resulting dataframe count will be same as that of input dataframe.

Usually look for scenarios where we have groupBy followed by join and this can be replaced with window.

org.apache.spark.sql.expressions.Window

val wndw = Window.partitionBy\($"col1",$"col2"\).orderBy\($"col3".desc\).rowsBetween\(start=10,end=-10\)

group column col1 and col2 and for those group order by col3 descending and then for every row in the group select rows\(for watever function you want to do on wndw\) which are -10 above it and +10 below it.current row is 0. Value for every record will be given.

rowsBetween by default takes start as Window.currentRow and end as Window.unboundedFollowing\(which is nothing but Long.MaxValue\). We also can use Window.unboundedPreceding\(Long.MinValue\).

If you want to take the whole table as one group then:

`val wndw1 = Window.orderBy($"age").rowsBetween(start=Window.unboundedPreceding,end=Window.unboundedFollowing)`

df.select\($"\*", **max\($"col99"\).over\(wndw\)**.as\("drvd"\)\)

Now as you see we can use all existing aggregate functions like max,min,sum,count etc .

Along with that we can also use :

Ranking Functions : rank,dense\_rank,percent\_rank,ntile,row\_number

Diff between rank vs dense\__rank vs row_\_number

| Name | rank | dense\_rank | row\_number |
| :--- | :--- | :--- | :--- |
| vin | 1 | 1 | 1 |
| she | 2 | 2 | 2 |
| she | 2 | 2 | 3 |
| she | 2 | 2 | 4 |
| nam | 5 | 3 | 5 |

** Ranking Function\(except ntile\) has pre-requisties that it should have a "orderBy" and also if present then rowsBetween should have start as Window.unBoundedPreceding and end to be Window.currentRow which logically makes sense..**

**First the grouping happens,then the ordering is available ,then the rowsBetween and finally the agg function**

percent\__rank is \(rank -1\)/\(total\_rank -1\) ,here total\_rank is per group and percent\_rank is always between 0 and 1._

ntile is non-determinsitic. Basically it will take the group created from partitionBy and orderBy and then splits then into ntile\(n\) ie n group ids. say we have 47 recods in group and ntile number is given as 4, since 47 is not perfectly divisble by 4 ,so its gets splits into two groups with one difference ,so now u will have 12 1's,12 2's ,12 3's and 11 4's in the given orderBy =&gt; [&lt;LINK&gt;](https://docs.microsoft.com/en-us/sql/t-sql/functions/ntile-transact-sql)

Analytic Functions  : lag, lead are the ones I am mainly concerned about. **rowsBetween and rangeBetween does NOT work here and we always require orderBy to be there which logically makes sense.**

`val wndw = Window.partitionBy($"city").orderBy($"age")`

`df1.select($"*",lag($"id",1).over(wndw).as("lag")).show()`

Part of the CDC Type 2 Logic :

```
scala> case class Info(id:Int,name:String,start_dt:java.sql.Timestamp,end_dt:java.sql.Timestamp)
defined class Info


scala> val data = List(Info(1,"vin",Timestamp.valueOf("2017-07-31 00:00:00"),Timestamp.valueOf("2099-12-31 00:00:00")),Info(1,"Vinyas",Timestamp.valueOf("2017-08-31 00:00:00"),Timestamp.valueOf("2099-12-31 00:00:00")))
data: List[Info] = List(Info(1,vin,2017-07-31 00:00:00.0,2099-12-31 00:00:00.0), Info(1,Vinyas,2017-08-31 00:00:00.0,2099-12-31 00:00:00.0))

scala> val wndw = Window.partitionBy($"id").orderBy($"start_dt")
wndw: org.apache.spark.sql.expressions.WindowSpec = org.apache.spark.sql.expressions.WindowSpec@26dc1405

scala> val ds1 = spark.createDataFrame(data)
ds1: org.apache.spark.sql.DataFrame = [id: int, name: string ... 2 more fields]

scala> ds1.select($"id",$"name",lead($"start_dt" - expr("INTERVAL 1 DAY"),1,"2099-12-31 00:00:00").over(wndw))
res78: org.apache.spark.sql.DataFrame = [id: int, name: string ... 1 more field]

scala> ds1.select($"id",$"name",lead($"start_dt" - expr("INTERVAL 1 DAY"),1,"2099-12-31 00:00:00").over(wndw)).show
+---+------+------------------------------------------------------------------------------------------------------------------------+
| id|  name|lead((start_dt - interval 1 days), 1, 2099-12-31 00:00:00) OVER (PARTITION BY id ORDER BY start_dt ASC UnspecifiedFrame)|
+---+------+------------------------------------------------------------------------------------------------------------------------+
|  1|   vin|                                                                                                    2017-08-30 00:00:...|
|  1|Vinyas|                                                                                                    2099-12-31 00:00:...|
+---+------+------------------------------------------------------------------------------------------------------------------------+


scala> ds1.select($"id",$"name",$"start_dt",$"end_dt",lead($"start_dt" - expr("INTERVAL 1 DAY"),1,"2099-12-31 00:00:00").over(wndw).as("new_end_dt")).show(false)
+---+------+---------------------+---------------------+---------------------+
|id |name  |start_dt             |end_dt               |new_end_dt           |
+---+------+---------------------+---------------------+---------------------+
|1  |vin   |2017-07-31 00:00:00.0|2099-12-31 00:00:00.0|2017-08-30 00:00:00.0|
|1  |Vinyas|2017-08-31 00:00:00.0|2099-12-31 00:00:00.0|2099-12-31 00:00:00.0|
+---+------+---------------------+---------------------+---------------------+

//The default value i passed was a string,but spark was smart enough converted to timestamp itself.ie why default is taken as Any
scala> ds1.select($"id",$"name",$"start_dt",$"end_dt",lead($"start_dt" - expr("INTERVAL 1 DAY"),1,"2099-12-31 00:00:00").over(wndw).as("new_end_dt")).printSchema
root
 |-- id: integer (nullable = false)
 |-- name: string (nullable = true)
 |-- start_dt: timestamp (nullable = true)
 |-- end_dt: timestamp (nullable = true)
 |-- new_end_dt: timestamp (nullable = true)
```

%%%%%%%%% EDIT  RANGE\_BETWEEN %%%%%%%%%%%%%%

## DataFrameReader

Some Important points while reading delimited text file:

    spark.read ==> Will return a DataFrameReader


    <li>`sep` (default `,`): sets a single character as a separator for each
       * field and value.</li>
       * <li>`encoding` (default `UTF-8`): decodes the CSV files by the given encoding
       * type.</li>
       * <li>`quote` (default `"`): sets a single character used for escaping quoted values where
       * the separator can be part of the value. If you would like to turn off quotations, you need to
       * set not `null` but an empty string. This behaviour is different from
       * `com.databricks.spark.csv`.</li>
       * <li>`escape` (default `\`): sets a single character used for escaping quotes inside
       * an already quoted value.</li>

       spark.read.option("sep","~").option("quote","|").csv("<hdfs location>")

Different Read modes in Spark:

PERMISSIVE , DROPMALFORMED ,FAILFAST .Default is PERMISSIVE ,but behaviour of PERMISSIVE is not what i expected ,see below:

```
bash-4.1$ hadoop fs -cat /user/test1/test1.txt
1,vin,23.34,shet
2,nam,hj,aj
bash-4.1$

scala> val sch = StructType(StructField("id",IntegerType)::StructField("fname",StringType)::StructField("amt",DoubleType)::StructField("lname",StringType)::Nil)
sch: org.apache.spark.sql.types.StructType = StructType(StructField(id,IntegerType,true), StructField(fname,StringType,true), StructField(amt,DoubleType,true), StructField(lname,StringType,true))


//I expected below to WORK but it FAILS:
scala> val df1 = spark.read.schema(sch).option("mode","PERMISSIVE").format("csv").load("/user/test1/test1.txt")
df1: org.apache.spark.sql.DataFrame = [id: int, fname: string ... 2 more fields]



scala> df1.show()
18/03/26 15:24:11 ERROR Executor: Exception in task 0.0 in stage 4.0 (TID 4)
java.text.ParseException: Unparseable number: "hj"
REASON this fails is its a parsing issue.

********************
PERMISSIVE vs FAILFAST works when we have missing data ie your schema says you have 5 columns but a row has only
two columns.
If there is a parsing issue ie your schema says int and it has character then it fails irrespective of the mode.
*******************************

//Below WORKS:

scala> val df1 = spark.read.schema(sch).option("mode","DROPMALFORMED").format("csv").load("/user/test1/test1.txt")
df1: org.apache.spark.sql.DataFrame = [id: int, fname: string ... 2 more fields]

scala> df1.show()
18/03/26 15:26:33 WARN CSVRelation: Parse exception. Dropping malformed line: 2,nam,hj,aj
+---+-----+-----+-----+
| id|fname|  amt|lname|
+---+-----+-----+-----+
|  1|  vin|23.34| shet|
+---+-----+-----+-----+


bash-4.1$ hadoop fs -cat /user/test1/test1.txt
1,vin,23.34
2,nam,1.2,aj
bash-4.1$

scala> val df1 = spark.read.schema(sch).option("mode","DROPMALFORMED").format("csv").load("/user/test1/test1.txt")
df1: org.apache.spark.sql.DataFrame = [id: int, fname: string ... 2 more fields]

scala> df1.show()
18/03/26 15:31:58 WARN CSVRelation: Dropping malformed line: 1,vin,23.34
+---+-----+---+-----+
| id|fname|amt|lname|
+---+-----+---+-----+
|  2|  nam|1.2|   aj|
+---+-----+---+-----+


//HERE PERMISSIVE WORKS AS EXPECTED,but FAILFAST would throw exception
scala> val df1 = spark.read.schema(sch).option("mode","PERMISSIVE").format("csv").load("/user/test1/test1.txt")
df1: org.apache.spark.sql.DataFrame = [id: int, fname: string ... 2 more fields]

scala> df1.show()
+---+-----+-----+-----+
| id|fname|  amt|lname|
+---+-----+-----+-----+
|  1|  vin|23.34| null|
|  2|  nam|  1.2|   aj|
+---+-----+-----+-----+
```

```
spark.read      //DataFrameReader
.option("","") // options like below
.option("","")  // like header->true,quotes->",sep->,
.schema(sch:StructType)
.format("")  //like parquet,csv,json,text
.load("")  //Path.Can be directory or file.Understands partition also


Points about Date:
bash-4.1$ hadoop fs -cat /user/test1/info.txt
1,vin,31/7/2018
2,she,21/8/2018
bash-4.1$

scala> val sch = StructType(StructField("id",IntegerType)::StructField("fname",StringType)::StructField("dt",DateType)::Nil)
sch: org.apache.spark.sql.types.StructType = StructType(StructField(id,IntegerType,true), StructField(fname,StringType,true), StructField(dt,DateType,true))

//Look at the dateFormat i have given,by default spark expects date to be in format yyyy-MM-dd .
//Similarly be careful on TimeStamp also.
scala> val df1 = spark.read.schema(sch).option("mode","DROPMALFORMED").option("dateFormat","dd/MM/yyyy").format("csv").load("/user/test1/info.txt")
df1: org.apache.spark.sql.DataFrame = [id: int, fname: string ... 1 more field]

scala> df1.show
+---+-----+----------+
| id|fname|        dt|
+---+-----+----------+
|  1|  vin|2018-07-31|
|  2|  she|2018-08-21|
+---+-----+----------+
```

## DataFrameWriter

```
df1.write -> DataFrameWriter
.option("compression","snappy") //This has the highest preference see default below
.mode()   //Wonder why the mode is explicit in DataFrameWriter
.format("")
.partitionBy("","")
.save("")

/*
Different modes are 
scala> import org.apache.spark.sql.SaveMode._
import org.apache.spark.sql.SaveMode._

Append
OverWrite
Ignore
ErrorIfExists


df.write.format("parquet").option("path","").partitionBy("","").saveAsTable("") 

//Compression is based on SparkConf value : spark.sql.parquet.compression.codec/spark.sql.orc.compression.codec 
//,ie iF not given in option like above
```

Aggregate Functions : It accepts a group of records.These are the ones which can work on RelationalGroupedDataset ie returned by groupBy methods.

```
scala> spark.conf.get("spark.sql.retainGroupColumns")
res32: String = true

Since this is set to true ,after the agg on grouped dataframe ,the grouped column is retained by default
```

```
/user/vin.txt is non existing location.RDD based read is lazy whil dataframe is not.
But dataframe does NOT still load the data and its lazy.


scala> sc.textFile("/user/vin.txt")
res0: org.apache.spark.rdd.RDD[String] = /user/vin.txt MapPartitionsRDD[1] at textFile at <console>:25

scala> spark.read.parquet("/user/vin.txt")
org.apache.spark.sql.AnalysisException: Path does not exist: hdfs://nn1/user/vin.txt;
  at
```

```
spark.sql.parquet​.filter​Pushdown -> default to True.
skips unnecessary rows where id> 100

spark.read.format("parquet").load(/location).select($"col1",$"col2").filter($"id" > 100)
```

```
**Create DataFrame from RDD of UserDefined Objects**

scala> case class Person(id:Int,age:Int,fname:String,lname:String)
defined class Person

scala> val r1 = sc.makeRDD(List(Person(1,30,"sam","marx"),Person(2,65,"vladmir","putin"),Person(3,63,"angela","merkel")))
r1: org.apache.spark.rdd.RDD[Person] = ParallelCollectionRDD[0] at makeRDD at <console>:26

scala> spark.createDataFrame(r1)
res0: org.apache.spark.sql.DataFrame = [id: int, age: int ... 2 more fields]

scala> val df1 = spark.createDataFrame(r1)
df1: org.apache.spark.sql.DataFrame = [id: int, age: int ... 2 more fields]

scala> df1.show()
+---+---+-------+------+
| id|age|  fname| lname|
+---+---+-------+------+
|  1| 30|    sam|  marx|
|  2| 65|vladmir| putin|
|  3| 63| angela|merkel|
+---+---+-------+------+


scala> val df2 = r1.toDF()
df2: org.apache.spark.sql.DataFrame = [id: int, age: int ... 2 more fields]

scala> df2.show()
+---+---+-------+------+
| id|age|  fname| lname|
+---+---+-------+------+
|  1| 30|    sam|  marx|
|  2| 65|vladmir| putin|
|  3| 63| angela|merkel|
+---+---+-------+------+

scala> import org.apache.spark.sql.types._
import org.apache.spark.sql.types._

scala> import org.apache.spark.sql.Row
import org.apache.spark.sql.Row

scala> val r2 = r1.map(r => Row(r.id,r.fname))
r2: org.apache.spark.rdd.RDD[org.apache.spark.sql.Row] = MapPartitionsRDD[7] at map at <console>:32

scala> val sch = StructType(StructField("id",IntegerType)::StructField("name",StringType)::Nil)
sch: org.apache.spark.sql.types.StructType = StructType(StructField(id,IntegerType,true), StructField(name,StringType,true))

scala> val df3 = spark.createDataFrame(r2,sch)
df3: org.apache.spark.sql.DataFrame = [id: int, name: string]

scala> df3.show()
+---+-------+
| id|   name|
+---+-------+
|  1|    sam|
|  2|vladmir|
|  3| angela|
+---+-------+
```

DataFrame to RDD :

```
scala> case class Per(id:Int,fname:String)
defined class Per

scala> df1.rdd.map(row => Per(row.getInt(0),row.getString(2)))
res7: org.apache.spark.rdd.RDD[Per] = MapPartitionsRDD[16] at map at <console>:37

scala> val prdd = df1.rdd.map(row => Per(row.getInt(0),row.getString(2)))
prdd: org.apache.spark.rdd.RDD[Per] = MapPartitionsRDD[17] at map at <console>:36

scala> spark.createDataFrame(prdd)
res8: org.apache.spark.sql.DataFrame = [id: int, fname: string]

scala> spark.createDataFrame(prdd).show()
+---+-------+
| id|  fname|
+---+-------+
|  1|    sam|
|  2|vladmir|
|  3| angela|
+---+-------+
```

**BroadCast Join : **

```
scala> spark.conf.get("spark.sql.autoBroadcastJoinThreshold")
res52: String = 10485760

So if the data of one of the table you joining is less the above then spark will broadcast that table by itself.
```

To get the memory file Programmatically,can be used only with RDD:

```
scala> val r2 = df.rdd
r2: org.apache.spark.rdd.RDD[org.apache.spark.sql.Row] = test MapPartitionsRDD[13] at rdd at <console>:25

scala> r2.name = "test100"
r2.name: String = test100

scala> r2.cache
res14: r2.type = test100 MapPartitionsRDD[13] at rdd at <console>:25

scala> r2.count
res15: Long = 224366121

scala>  sc.getRDDStorageInfo.map(_.name).length
res16: Int = 2

scala>  sc.getRDDStorageInfo.filter(_.name == "test100").map(_.memSize)
res22: Array[Long] = Array(8627743768)
```

# Joins:

**shuffled hash** join is the default join.



