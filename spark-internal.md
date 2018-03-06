SparkSession is a companion object which has the builder method which returns a Builder Object,.

Builder has fields : options which is a mutable HashMap\[String,String\] ,  extension:SparkExtensions object , userSuppliedContext:Option\[SparkContext\] which set to None

SparkExtensions =&gt; This seems to be the the place for all the Rules Builders for different Plans.Will come back to this.

Builder class also has methods as appName,master,config\(k,v\).

So Fluent Interface is used to create a SparkSession object ie We have builder method inside SparkSession Compannion object ,then when we call a builder method a Builder object is returned then we can use this to set various values using the setter method which sets the value\(using a HashMap ,options\) and returns the builder object ,finally when we call a getOrCreate method in builder object ,this will create a SparkSession object using the options HashMap.

SparkSession needs a SparkContext , Option\[SharedState\], Option\[SessionState\] , SparkSessionExtensions



