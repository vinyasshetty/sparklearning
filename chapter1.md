# Spark Key Value RDD

* Spark has its PairRDDFunctions class which has all the functions which can be used on Pair RDD's.Its made available via implicits. &lt;NEED TO UNDERSTAND BELOW IMPLICIT AND TYPE CONCEPT&gt;
* `class PairRDDFunctions[K, V](self: RDD[(K, V)]) (implicit kt: ClassTag[K], vt: ClassTag[V], ord: Ordering[K] = null) extends Logging with Serializable {`
* **OrderedRDDFunctions** class has the ordering related method for RDD\[\(K,V\)\] where ordering for keys K should be available available.Functions: sortByKey, repartitionAndSortWithinPartitions and filterByRange.For most of the basic types ordering of K types are already available but if you have complex K type then u have to give the Ordering explicitly
* These methods in PairRDDFunctions and OrderedRDDFunctions are the expensive ones since they include wide transformations.
* They can cause different types of errors like : Out of Memory Error at Driver, Out of Memory Error at Exceutor, Shuffling Failure and Tasks straggling due to large compute time.
* Out of Memory Error at Driver is caused mainly due to action collecting lots of data at driver others are caused due to shuffling related.
* Always SHUFFLE LESS AND SHUFFLE BETTER.
* Advantage of Key Value type data comes from the fact that each key and its corresponding value can be calcualted in parallel.
* **flatMap** is very useful method it can be used to as a combination of map+filter and also can be used to increade the count of RDD elements.
* **df.rdd.flatMap\(row =&gt; Range\(0,df.columns.length\).map\(x=&gt;\(x,row.get\(x\)\)\)**
* lookup is a action on RDD\[\(K,V\)\] and returns a Seq\[V\] . lookup causes a shuffle if the rdd is NOT partitioned 
* Actons usually move data out of the executors into either driver or to some external target like hdfs.
* countByKey,lookup,collect,collectAsMap are all actions.
* **Key/value transformations can also cause memory errors, most often in the executors,if they require all the data associated with one key to be kept in memory on one partition.**
* **Some actions causes a shuffle depending of the last rdd was partitioned or no. Ex : lookup\(k\),countByKey\(\)**
* One of the dangerous function is groupByKey because this will make all the values belonging to a key to be avaiable in exceutor memory at once and if it cant fit then it causes OOM in Executor.
* aggregateByKey,combineByKey in these operators a "combining" of values belonging to the same happens once on the map side also,hence the number of values for a given key to shuffle is less.
* Most of the bykey operations is built on "combineByKey" 
* combineByKey\(f:V=&gt;U\)\(f1:\(U,V\)=&gt;U,f2:\(U,U\)=&gt;U\):RDD\[\(K,U\)\]  ,f and f1 are run within a partition and f2 is run usig data from across partition.f is run when a new key's value has been found and f1 is run using the value when a key has been found again.f2 is run when all the vlues belonging to a given key are shuffled.
* aggregateByKey\(z:U\)\(f:\(U,V\)=&gt;U,f1:\(U,U\)=&gt;U\):RDD\[\(K,U\)\],same as combineByKey but iniitial is a value instead of a function.
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

* Main difference between reduceByKey/foldByKey vs combineByKey/aggregateByKey is reduce expects the output value type also to be same as the input value type while in combine the output value type needs to same as the intial value type set by us which can be different from the input dataset value type.See above the U and V types .Look at the method signature  in PairRDDFunctions.scala.

* **CombineByKey and all of the aggregation operators built on top of it \(reduceByKey, foldLeft, foldRight, aggregateByKey\) are no better than groupByKey in terms of memory errors if they cause the accumulator to become too large for one key. In fact, if you look up the implementation of groupByKey, you can see that it is actually implemented using combineByKey where the accumulator is an iterator with all the data. Thus, the accumulator is the size of all the data for that key. In other words, these operations are unlikely to cause memory errors as long as the combining steps make the data smaller. However, if the accumulator gets larger with the addition of each new record, it will eventually cause memory errors if there are many records associated with one key.**

* Beyond being less likely to run out of memory than groupByKey, the following four functions—reduceByKey, treeAggregate, aggregateByKey, and foldByKey—are implemented to use map-side combinations, meaning that records with the same key are combined before they are shuffled. This can greatly reduce the shuffled read.

## Multiple RDD Operations

As much of the byKey operatios are implemented using combineByKey, much of the join operations are implemented using cogroup operations. Returns a CoGroupedRDD type.

PairRDDFunctions provides several implementations of co-group methods and it can take  two or three RDD's to co group.It expects all of them to have the same Key Type but the Value types can be different.This also follows the same default partitioner logic.CoGroup returns a RDD of K and a tuple of Iterable values.Size of tuple can be two or three based on \# of RDD cogrouped.

Cogroup also has the same problem like groupByKey ie if values belonging to one key cannot fit in memory then OOM error.

Some narrow transformations like mapValues preserve the partitioning.

## Partitioning

* **Hashpartitioning **does the hashing of the keys and determines to which partition the key and its value should go.This requires the key and the number of partitions which is determined using the deafultPartition method.
* **RangePartitioning** : Here every rdd partition is sampled to determine the range of keys and for equal optimized distribution .Based on that each key and its value are sent to a partition based on the range of the partition.Now we need to sort the partitions and now the whole rdd gets sorted by key.This is used for sorting.

* Creating a RangePartitioner not only requires the number of partitions but also required the RDD of key value type ,so that sampling can be done on keys and it expects keys to have Ordering defined.SInce sampling of RDD needs to be done RangePartitioner is slower then HashPartitioner.

* | RangePartitioner | HashPartitioner |
  | :--- | :--- |
  | Requires Number of partitions and also RDD | Requires Number of partitions |
  | Expects,the Key\(K\) to have Ordering implemented in RDD\[\(K,V\)\] | K can be of any types |
* **CustomPartitioning** : This can be done by extending Partitioner class and implementing numPartitions:Int  and getPartition\(key:Ant\):Int  .  Optional methods =&gt; equals\(other:Any\):Boolean and hashcode\(\):Int

* **Concept of Partitioner comes into picture only for RDD of Key Value Pair Type.**

* When we do a wide transformation on RDD\[\(K,V\)\] like cogroup , combineByKey,sortByKey etc then the resulting RDD will always have a partitioner.What sort of Partitioner and \# of partitions is set depends on teh defaultPartition method in Partitioner object.

* rdd.partitioner will return a Option\[Partitioner\] ,if we use some transformation which has the potential of changing the key then the resulting RDD  will lose its partitioner and will become None. =&gt; **com.acc.vin.MaintainPartitions**

* mapParitions and mapParitionsWithIndex can preserve partitions if preservesPartitioning is set to true ,default its false.

* When using two or more RDD's ,think about co-location and co-partitioning of RDD's.

* **RDD's can be co-located if RDD's have the same Partitioner  and data associated with them is in the same excutor.This will happen if they partitioned during the lineage of the same job. ==&gt; com.acc.vin.CoLocated **

* ** “same partitioner” means the partitioner objects are equal according to the equality function defined in the partitioner class.So this basically means for HashPartitioner the number of partitions also should be same.**

* **Co-Partitioned RDD's mean that two rdds have been partitioned using the same partitioner but as a part of two different jobs,now this means that when you join them it does NOT need a full shuffle but the partitions needs to be aligned so there will be some network transfer.=&gt;com.acc.vin.CoPartitioned**

* rdd.keys will return RDD\[K\],here keys will not be distinct if input has duplicate keys, we also have rdd.values=&gt; RDD\[V\]

* OrderedRDDFunctions =&gt; sortByKey returns a RDD with RangePartitioner and also for sortByKey,it expects key to have implemented Ordered ,now most of the the regular scala basic types like Int,STring Double etch have already Ordered implemented.Also scala has **Tuple2 **type Ordered implemented so if you have a RDD\[\(\(K1,K2\),V\)\] works fine if you sortByKey=&gt; ** com.acc.vin.SortByKeyTupl2 **

* We can sort Keys in RDD by first repartitioning using RangePartitioning and then use mapPartitions to sort the Key data,but internally spark's sortByKey is more efficient since it sorts within the shuffle stage onto the individaul machines.

* SecondarySort is a technique  where if you want sort a value along with a key ,then you make the key as composite and then sort this composite key.This technique is called SecondarySort.To implement this in spark we have the reprtitionAndSortWithinPartitions functions.This is a wide transformation and it takes a Partitioner object and implicit Ordering of the keys of the RDD.

* ** If we are using hash partitioning, this function does not actually sort values by the first key. Rather, it groups keys with the same hash value on the same machine. Thus, if we run the function of the values one through five and use four partitions, the first partition will contain one and five. To force the keys to appear in true sorted order, we would need to define a range partitioner.**



