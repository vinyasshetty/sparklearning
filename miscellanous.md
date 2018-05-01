* Aws Redshift has a master slave cluster architecture.
* Master Node plans ,co-ordinates and oversees query execution submitted by BI client like Tableau,microstrategy etc.
* Only the Master Node is a Postgre sql cluster which has all the metadata information which helps in optimizing the queries.
* DataLoad is parallel and its most efficient when loaded from s3 ,but s3 needs to have multiple flies.

* Also data is replicated  in redshift and when you instantiate 100 gb redshift cluster,you get raw 100 gb space without taking into account the replication.



