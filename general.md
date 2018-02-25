* Partition corresponds to one task in Spark..This is the basic unit of Parallelism.

* A Rdd without partition ,in that data is assigned based on partition size and data size.
* A Rdd with partition will gurantee that either a value beloging to a given will always end up same partition or a range of keys will always go to a same partition.
* ** transformations with narrow dependencies as those in which “each partition of the parent RDD is used by at most one partition of the child RDD.” The creators define transformations with wide dependencies as transformations in which “multiple child partitions may depend on \[each partition in the parent\].” **
* Within a given stage the number of tasks used to complete a computation corresponds to each output partition rather than each input partition—when RDDs are evaluated; the tasks needed to compute a transformation are computed on the child partitions. ie say if i have a multi step narrow transformations where input/first-parent partition count is 200 and then a coalesce\(to 50\) and a action. Now this stage will run with just 50 tasks and not 200.
* master can be given to run on local mode with avialable cores "local\[\*\]" .

* Usually in Prod master is set to "yarn" ie if using yarn cluster manager.

* deploy-mode is either client or cluster.



