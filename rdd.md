** Internally, each RDD is characterized by five main properties:**

* A list of partitions  =&gt;  protected def getPartitions: Array\[Partition\]
* A function for computing each split =&gt; def compute\(split: Partition, context: TaskContext\): Iterator\[T\]
* A list of dependencies on other RDDs =&gt; protected def getDependencies: Seq\[Dependency\[\_\]\] = deps
* Optionally, a Partitioner for key-value RDDs \(e.g. to say that the RDD is hash-partitioned\)
* Optionally, a list of preferred locations to compute each split on \(e.g. block locations for an HDFS file\)

Dependency abstract class has just one abstract method called rdd.

It is implemented by abstract class NarrowDependency.It has two method

def getParents\(partitionId: Int\): Seq\[Int\]

override def rdd: RDD\[T\] = \_rdd

These extends NarrowDependency : OneToOneDependency and RangeDependency. ShuffleDependency extends from Depedency.

* repartition and coalesce is used to change the partition count.repartition is implemented using coalesce.
* If you have to reduce the number of partitions then we use coalesce \(though this is a type of wide transformation it does not shuffle since we dont have to know about data in the partition\) but repartition will cause a shuffle and it will use HashPartitioner to shuffle.
* rdd.partitioner will return a Option\[Partitioner\] ,so we can use isDefined to check if Rdd has a partitioner set.



