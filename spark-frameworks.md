## SCD Type 2

In databases ,there is a implementation of Slow Changing Dimension Type 2 .So below will help us to implement the same in Spark.

I am implementing SCD Type 2.

```
Say we have a Actual Data(Current) like below :

+---+--------+------+-----+-----+---------------------+---------------------+
|id |fname   |lname |zip  |amt  |start_dt             |end_dt               |
+---+--------+------+-----+-----+---------------------+---------------------+
|1  |vinyas  |shetty|28213|32.23|2017-02-21 00:00:00.0|2018-03-28 00:00:00.0|
|1  |vinyas  |shetty|60606|23.34|2018-03-29 00:00:00.0|2099-12-31 00:00:00.0|
|2  |namratha|rao   |2811 |22.32|2018-03-31 00:00:00.0|2099-12-31 00:00:00.0|
|3  |harold  |finc  |2877 |56.43|2018-05-01 00:00:00.0|2099-12-31 00:00:00.0|
+---+--------+------+-----+-----+---------------------+---------------------+


SnapShot Data(ie Latest Data) :
+---+--------+------+-----+-----+---------------------+---------------------+
|id |fname   |lname |zip  |amt  |start_dt             |end_dt               |
+---+--------+------+-----+-----+---------------------+---------------------+
|1  |vinyas  |shetty|60611|23.22|2018-04-01 00:00:00.0|2099-12-31 00:00:00.0|
|2  |namratha|shetty|2811 |30.23|2018-04-01 00:00:00.0|2099-12-31 00:00:00.0|
+---+--------+------+-----+-----+---------------------+---------------------+

End Result :
+---+--------+------+-----+-----+---------------------+---------------------+
|id |fname   |lname |zip  |amt  |start_dt             |end_dt               |
+---+--------+------+-----+-----+---------------------+---------------------+
|1  |vinyas  |shetty|28213|32.23|2017-02-21 00:00:00.0|2018-03-28 00:00:00.0|
|1  |vinyas  |shetty|60606|23.34|2018-03-29 00:00:00.0|2018-03-31 23:00:00.0|
|1  |vinyas  |shetty|60611|23.22|2018-04-01 00:00:00.0|2099-12-31 00:00:00.0|
|2  |namratha|rao   |2811 |22.32|2018-03-31 00:00:00.0|2018-03-31 23:00:00.0|
|2  |namratha|shetty|2811 |30.23|2018-04-01 00:00:00.0|2099-12-31 00:00:00.0|
|3  |harold  |finc  |2877 |56.43|2018-05-01 00:00:00.0|2099-12-31 00:00:00.0|
+---+--------+------+-----+-----+---------------------+---------------------+
```

Application conf in resources directory :

```
cdc-type2 {
  start_dt="start_dt"
  end_dt="end_dt"
  end_value="2099-12-31 00:00:00"
  unique_columns="id"
  end_dt_logic="1 HOUR"
  history_path="<hdfs_location>"
  snapshot_path="<hdfs_location>"
  output_path="<hdfs_location>"
  format="parquet"
}
```

Certain Rules/Pre- Requisitities:

```
Schema of history and snapshot data should be same else will fail
history should NOT have dups (ie for a given unique columns there should be only one record where end_dt is end_value).
snapshot should NOT have dups (ie for a given unique columns there should be only one record where end_dt is end_value).

end_dt_logic : basically subtracts the new record start dt value with end_dt_logic value
               and puts that value as end_dt of the old record ,which needs to be updated.
               we can use MINUTE,HOUR,SECOND,DAY etc.
unique_column : columns which make the records unique,can be comma separated if multiple.Do NOT include start_dt end_dt columns.

If we have active record with same unique column,start_dt in both history and snapshot,then code 
will randomly pick one and drop the other because it will NOT know which is the correct one.
```

Code :

**&lt;Need to Provide GIT Code link&gt;**

```
package com.vin.cdc

import com.typesafe.config.ConfigFactory
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.types._
import com.vin.cdc.cdcType2.cdcType2Main
import org.apache.spark.sql.functions._

object cdcMain {

  def main(args:Array[String])={

    val spark = SparkSession.builder().appName("CDC").enableHiveSupport().getOrCreate()

    /*
    Input Data and SnapShot data expected to have same schema and order
     */
    val conf = ConfigFactory.load()
    val start_dt = conf.getString("cdc-type2.start_dt")
    val end_dt = conf.getString("cdc-type2.end_dt")
    val end_value = conf.getString("cdc-type2.end_value")
    val unique_columns = conf.getString("cdc-type2.unique_columns")
    val end_dt_logic = conf.getString("cdc-type2.end_dt_logic")
    val format = conf.getString("cdc-type2.format")
    val hist_path = conf.getString("cdc-type2.history_path")
    val snap_path = conf.getString("cdc-type2.snapshot_path")
    val opt_path = conf.getString("cdc-type2.output_path")


    val hist = spark.read.format(format).load(hist_path)
    val snap = spark.read.format(format).load(snap_path)

    println("INPUT DF1 :" )
    hist.show()

    println("INPUT DF2 :")
    snap.show()

    val final_res = cdcType2Main(spark,hist,snap,start_dt,end_dt,end_value,unique_columns,end_dt_logic)

    final_res.write.mode("overwrite").format("parquet").save(opt_path)

  }

}
```

```
package com.vin.cdc


import org.apache.spark.sql.expressions.Window
import org.apache.spark.sql.functions._
import org.apache.spark.sql.types.TimestampType
import org.apache.spark.sql.{Column, DataFrame, SparkSession}

object cdcType2 {
  def cdcType2Main(spark:SparkSession, hist:DataFrame, snap:DataFrame,
                   start_dt:String,end_dt:String,end_value:String,unique_columns:String,end_dt_logic:String):DataFrame={

    validation(hist,snap,start_dt,end_dt,unique_columns)

    val end_value_ts = lit(end_value).cast(TimestampType)

    val (active_records,closed_records) = splitHistActiveClosed(hist,start_dt,end_dt,end_value_ts)

    println("ACTIVE Records ")
    active_records.show(false)

    println("CLOSED RECORD ")
    closed_records.show(false)

    //Just a extra check to make sure snapshot has only active records
    val active_snap = snap.filter(snap(end_dt) <=> end_value_ts)
    val active_with_cdc = cdcLogicImpl(spark,active_records,active_snap,unique_columns,
      start_dt,end_dt,end_value,end_dt_logic)

    println("ACTIVE WITH CDC ")
    active_with_cdc.show(false)

    val final_result = active_with_cdc.union(closed_records)
    final_result

  }


  def validation(hist:DataFrame,snap:DataFrame,
                 start_dt:String,end_dt:String,
                 unique_columns:String) ={
    val hist_schema = hist.schema
    val snap_schema = snap.schema
    val diff_schema = hist_schema.toSet.union(snap_schema.toSet).diff(hist_schema.toSet.intersect(snap_schema.toSet))
    if(!diff_schema.isEmpty){
      throw new Exception(
        s"""
           |Schema between History and SnapShot Did Not Match
           |History : ${hist_schema}
           |Snapshot : ${snap_schema}
           |Difference : ${diff_schema}
         """.stripMargin)
    }
    //We will let Spark throw the error java.lang.IllegalArgumentException if element NOt found
    val dummy_val = hist_schema(Set(start_dt,end_dt) ++ unique_columns.split(",").toSet)

  }

  def splitHistActiveClosed( hist:DataFrame,
                             start_dt:String,end_dt:String,end_value_ts:Column):(DataFrame,DataFrame)={

    val active_records = hist.filter(hist(end_dt) <=>  end_value_ts)
    val closed_records = hist.filter(!(hist(end_dt) <=> end_value_ts))
    (active_records,closed_records)
  }

  def cdcLogicImpl(spark:SparkSession,active_records:DataFrame,snap:DataFrame,
                   unique_columns:String,start_dt:String,end_dt:String,
                   end_value:String,end_dt_logic:String):DataFrame={
    import spark.implicits._

    /*Need to think about this,do we fail when we have two active records with same start_dt,
     currently it just just drops one of them
     */
    val union_data = active_records.union(snap)
      .dropDuplicates(start_dt :: end_dt :: unique_columns.split(",").map(x=>x.trim).toList)

    val uniq = unique_columns.split(",").map(x=> col(x))

    val wndw = Window.partitionBy(uniq:_*).orderBy(col(start_dt).asc)

    //Code will fail if actual end_dt column is also called as "new_end_date".Need to look at handling end_value null
    val cdc_data = union_data.select($"*",
      (lead(col(start_dt) - expr(s"INTERVAL ${end_dt_logic}"),1,end_value).over(wndw) ).as("new_end_date"))

    cdc_data.drop(end_dt).withColumnRenamed("new_end_date",end_dt)

  }

}
```



