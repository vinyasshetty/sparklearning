# Spark Key Value RDD

* Spark has its PairRDDFunctions class which has all the functions which can be used on Pair RDD's.Its made available via implicits. &lt;NEED TO UNDERSTAND BELOW IMPLICIT AND TYPE CONCEPT&gt;
* `class PairRDDFunctions[K, V](self: RDD[(K, V)]) (implicit kt: ClassTag[K], vt: ClassTag[V], ord: Ordering[K] = null) extends Logging with Serializable {`
* OrderedRDDFunctions class has the ordering related method for RDD\[\(K,V\)\] where ordering for keys K should be available available.Functions: sortByKey, repartitionAndSortWithinPartitions and filterByRange.For most ofthe basic types ordering of K types are already available but if you have complex K type then u have give the Ordering explicitly
* These methods in PairRDDFunctions and OrderedRDDFunctions are the expensive ones since they include wide transformations.
* They can cause different types of errors like : Out of Memory Error at Driver, Out of Memory Error at Exceutor, Shuffling Failure and Tasks straggling due to large compute time.
* Out of Memory Error at Driver is caused mainly due to action collecting lots of data at driver others are caused due to shuffling related.
* Always SHUFFLE LESS AND SHUFFLE BETTER.
* Advantage of Key Value type data comes from the fact that each key and its corresponding value can be calcualted in parallel.
* **flatMap** is very useful method it can be used to as a combination of map+filter and also can be used to increade the count of RDD elements.
* **df.rdd.flatMap\(row =&gt; Range\(0,df.columns.length\).map\(x=&gt;\(x,row.get\(x\)\)\)**
* Actons usually move data out of the executors into either driver or to some external target like hdfs.
* countByKey,lookup,collect,collectAsMap are all actions.
* **Key/value transformations can also cause memory errors, most often in the executors,if they require all the data associated with one key to be kept in memory on one partition.**
* One of the dangerous function is groupByKey because this will make all the values belonging to a key to be avaiable in exceutor memory at once and if it cant fit then it causes OOM in Executor.
* aggregateByKey,combineByKey in these operators a "combining" of values belonging to the same happens once on the map side also,hence the number of values for a given key to shuffle is less.
* Most of the bykey operations is built on "combineByKey" 
* combineByKey\(f:V=&gt;U\)\(f1:\(U,V\)=&gt;U,f2:\(U,U\)=&gt;U\)  ,f and f1 are run within a partition and f2 is run usig data from across partition.f is run when a new key's value has been found and f1 is run using the value when a key has been found again.f2 is run when all the vlues belonging to a given key are shuffled.
* aggregateByKey\(z:U\)\(f:\(U,V\)=&gt;U,f1:\(U,U\)=&gt;U\),same as combineByKey but iniitial is a value instead of a function.
* all the ByKey functions are overloaded into 3 types,where 1\)is just the function,2\) is it takes the function and the numPartitions:Int ,3\)is it takes the function and the partitioner:org.apache.spark.Partitioner.
* **ByKey and join operators in RDD ,the partitioner and partition count is selected using below method:**
* `def defaultPartitioner(rdd: RDD[_], others: RDD[_]*): Partitioner = {`  
    `val rdds = (Seq(rdd) ++ others)`  
    `val hasPartitioner = rdds.filter(_.partitioner.exists(_.numPartitions > 0))`  
    `if (hasPartitioner.nonEmpty) {`  
    `hasPartitioner.maxBy(_.partitions.length).partitioner.get`  
    `} else {`  
    `if (rdd.context.conf.contains("spark.default.parallelism")) {`  
    `new HashPartitioner(rdd.context.defaultParallelism)`  
    `} else {`  
    `new HashPartitioner(rdds.map(_.partitions.length).max)`  
    `}`  
    `}`

  `}`

* 


