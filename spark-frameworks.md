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


SnapShot Data :
+---+--------+------+-----+-----+---------------------+---------------------+
|id |fname   |lname |zip  |amt  |start_dt             |end_dt               |
+---+--------+------+-----+-----+---------------------+---------------------+
|1  |vinyas  |shetty|60611|23.22|2018-04-01 00:00:00.0|2099-12-31 00:00:00.0|
|2  |namratha|shetty|2811 |30.23|2018-04-01 00:00:00.0|2099-12-31 00:00:00.0|
+---+--------+------+-----+-----+---------------------+---------------------+
```

Application conf :

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
history should NOT have dups (ie for a given unique columns there should be only one record where end_dt is end_value)
end_dt_logic : basically subtracts the new record start dt value with end_dt_logic value
               and puts that value as end_dt of the old record ,which needs to be updated.
               we can use MINUTE,HOUR,SECOND,DAY etc.
unique_column : columns which make the records unique,can be comma separated if multiple.Do NOT include start_dt end_dt columns.

If we have active record with same unique column,start_dt in both history and snapshot,then code 
will randomly pick one and drop the other because it will NOT know which is the correct one.
```



