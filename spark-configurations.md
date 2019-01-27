Spark Executors -&gt; Runs on a JVM.One executor per JVM\(**--num-executors**\)

Spark Executor Memory \(**--executor-memory**\)

Spark Executor Cores \(**--executor-cores**\)

**We have say 6 Nodes with 16 cores each and 64Gb RM each.**

So total cores = 16 \* 6 = 96

So total memory = 64 \* 6 = 384gb

ApplicationMaster needs one core.

Its better to keep around 4 or 5 cores per executor, this is somewhat ideal for hdfs i/o throughput.

With 5 core per executor, now each node will have 16/5 ~ 3 executor and 63gb/3 ~  21gb.

So total executor = 3\*6 1= 18 ,now give one for AM ,so we will run with around 17 executors and around 20gb executor memory. 

