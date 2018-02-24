# Spark Key Value RDD

* Spark has its PairRDDFunctions class which has all the functions which can be used on Pair RDD's.Its made available via implicits. &lt;NEED TO UNDERSTAND BELOW IMPLICIT AND TYPE CONCEPT&gt;
* `class PairRDDFunctions[K, V](self: RDD[(K, V)]) (implicit kt: ClassTag[K], vt: ClassTag[V], ord: Ordering[K] = null) extends Logging with Serializable {`
* OrderedRDDFunctions class has the ordering related method for RDD\[\(K,V\)\] where ordering for keys K is available.
* These methods in PairRDDFunctions and OrderedRDDFunctions are the expensive ones since they include wide transformations.
* They can cause different types of errors like : Out of Memory Error at Driver, Out of Memory Error at Exceutor, Shuffling Failure and Tasks straggling due to large compute time.
* Out of Memory Error at Driver is caused mainly due to action collecting lots of data at driver others are caused due to shuffling related.
* Always SHUFFLE LESS AND SHUFFLE BETTER.
* Advantage of Key Value type data comes from the fact that each key and its corresponding value can be calcualted in parallel.
* **flatMap** is very useful method it can be used to as a combination of map+filter and also can be used to increade the count of RDD elements.





