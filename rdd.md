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
* **Using iterator-to-iterator transforms in mapPartitions prevents whole partitions from being loaded into memory.**
* An important way to optimize Spark jobs for both time and space is to stick to primitive types rather than custom classes. Although it may make code less readable, using arrays rather than case classes or tuples can reduce GC overhead.  Scala arrays, which are exactly Java arrays under the hood, are the most memory-efficient of the Scala collection types. Scala tuples are objects, so in some instances it might be better to use a two- or three-element array rather than a tuple for expensive operations. The Scala collection types in general incur a higher GC overhead than arrays
* **Narrow Transformation** =&gt; One parent partition can send data to only one child partition.
* **Wide Transformation **=&gt; Majority of the child partitions receieve data from all the parent partitions.



