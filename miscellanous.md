* Aws Redshift has a master slave cluster architecture.
* Master Node plans ,co-ordinates and oversees query execution submitted by BI client like Tableau,microstrategy etc.
* Only the Master Node is a Postgre sql cluster which has all the metadata information which helps in optimizing the queries.
* DataLoad is parallel and its most efficient when loaded from s3 ,but s3 needs to have multiple flies.

* Also data is replicated  in redshift and when you instantiate 100 gb redshift cluster,you get raw 100 gb space without taking into account the replication.

* Data when loaded goes directly into slave and it does not need to go through master.

* **Disk storage for each node/slave is split into \# of vcpus.These are called as slices**

* Slave Nodes have two classification : 1\) ** Dense Compute **2\)** Dense Storage **

* When you upload the files from s3 , the number of splits of files is ideal to be equal to the total number of slices  in your aws redshift cluster.

* Idea files size\(post-compressed\) between 1MB-1GB

* Records are distributed in Redshift in on of the 3 types:

  ```
  1)Even Distribution Type : Here the row counts is looked at and in round robin fashion ,data 
  in distributed into different slices of the slave nodes.Values of the columns are not taken into
   consideration.This is good when you use all the rows while querying and not using this for any
   joining.

  2)Key Distribution Type : Same Keys ie values that are of same column are put into together in one slice or 
  in on node.
  ```



