\* Internally, each RDD is characterized by five main properties:

* A list of partitions  =&gt;  protected def getPartitions: Array\[Partition\]
* A function for computing each split =&gt; def compute\(split: Partition, context: TaskContext\): Iterator\[T\]
* A list of dependencies on other RDDs =&gt; protected def getDependencies: Seq\[Dependency\[\_\]\] = deps
* Optionally, a Partitioner for key-value RDDs \(e.g. to say that the RDD is hash-partitioned\)
* Optionally, a list of preferred locations to compute each split on \(e.g. block locations for an HDFS file\)



Dependency abstract class has just one abstract method called rdd.

It is implemented by class NarrowDependency,OneToOneDependency,ShuffleDependency and RangeDependency.



