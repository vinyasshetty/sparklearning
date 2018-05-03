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

* Records are distributed in Redshift Nodes in on of the 3 types:

  ```
  1)Even Distribution Type : Here the row counts is looked at and in round robin fashion ,data 
  in distributed into different slices of the slave nodes.Values of the columns are not taken into
   consideration.This is good when you use all the rows while querying and not using this for any
   joining.

  2)Key Distribution Type : Same Keys ie values that are of same column are put into together in one slice or 
  in on node.

  3)All distribution type : Whole data is loaded into every node.This is very slow. Use for small tables which 
  will be used to join.
  ```

Now once i have launched a Redshift cluster .TO access that either i can use my computer and have sqlbench installed.

But i will try something different.I will launch a Ec2 cluster and attach Ec2 IAM Role Policy to have S3 and Redshift access to it.

Then i will ssh into ec2 using the pem file\(key value pair\) and then use ec2 instance to access s3 using aws cli or access redshift by installing pgsql .

```
psql -h <redshift_end_point> -U <user> -d <dbname> -p 5439

 CREATE TABLE part                                                                                                                                                (
  p_partkey     INTEGER NOT NULL,
  p_name        VARCHAR(22) NOT NULL,
  p_mfgr        VARCHAR(6) NOT NULL,
  p_category    VARCHAR(7) NOT NULL,
  p_brand1      VARCHAR(9) NOT NULL,
  p_color       VARCHAR(11) NOT NULL,
  p_type        VARCHAR(25) NOT NULL,
  p_size        INTEGER NOT NULL,
  p_container   VARCHAR(10) NOT NULL
);

\dt (to list tables)

\l (to list databases)

\d+ <tablename> (to describe table)

Load data from s3 to redshift:
copy  part from 's3://s3testemr/redshiftdata/part-csv.tbl' credentials 'aws_access_key_id=<>;aws_secret_access_key=<>' csv
null as '\000';

```



