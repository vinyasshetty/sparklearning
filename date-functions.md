```
import org.apache.spark.sql.types._
val data = List(
  ("2015-01-01 23:59:59", "2015-01-02 00:01:02",1),
  ("2015-01-02 23:00:00", "2015-01-02 23:59:59",2),
  ("2015-01-02 22:59:58", "2015-01-02 23:59:59",3))

val df1 = spark.createDataFrame(data).select($"_1".cast(TimestampType).as("start_time")
                                             ,$"_2".cast(TimestampType).as("end_time"),$"_3".as("id"))
df1.show(false)
```

```
import org.apache.spark.sql.functions._
val condition = (to_date($"start_time") === to_date($"end_time")) &&  ($"start_time" + expr("INTERVAL 1 HOUR") >= $"end_time")
df1.filter(condition).show

//to_date function gets just the date type and only the date part of it
df1.select(to_date($"start_time"),$"start_time").printSchema

//This is really helpful to add/subtract from date and timestamp :
df1.select($"start_time",$"start_time" + expr("INTERVAL 1 HOUR")).show

df1.select(to_date($"start_time") + expr("INTERVAL 2 DAY"),to_date($"start_time")).show

//expr("INTERVAL VALUE UNIT") .Available units are YEAR, MONTH, uDAY, HOUR, MINUTE,
// SECOND, MILLISECOND, and MICROSECOND
```

```
//add_months(Column,n:Int).WOrks on date and timestamp,but converts to Date.

df1.select(add_months($"start_time",2),add_months(to_date($"start_time"),2),$"start_time").show

//cuurent date
df1.select(current_date(),current_timestamp()).show(false)

//Always date is expected to be in yyyy-mm-dd format and this convert to below string format:

df1.select(date_format(lit("2010-05-21").as("col1"),"yyyy.dd.MM"),
          date_format($"start_time","yyyy.dd.MM")).show()
```

```
Always date is expected to be in yyyy-mm-dd format in spark
```

```

```



