Hbase-indexer
============
Requirements
------------

https://github.com/NGDATA/hbase-indexer.wiki.git<br>
Clone in Desktop<br>
The following software is required for running the HBase Indexer:

*HBase 0.94.x<br>
*Solr 4.x in cloud mode<br>
*ZooKeeper 3.x (required by the two above packages)<br>
All components can be run on a single machine, or they can be run on multiple machines on a cluster.

# Details
`HBase`<br>
HBase 0.94.x is the supported version of HBase for the HBase Indexer. It is recommended to use the version of HBase 0.94.2 that is bundled with Cloudera CDH 4.2. However, other versions of HBase 0.94.x may also work. CDH 4.2 is currently used for testing.
HBase should be configured to use HDFS as its filesystem -- HBase Indexer is not fully functional if the local filesystem implementation is used instead of HDFS.

`Solr`<br>
Solr 4.x is required for the HBase Indexer, and it must be configured to run in cloud mode. Solr 4.2.0 is currently used for development testing.

`ZooKeeper`<br>
It is recommended that the version of ZooKeeper that is bundled in CDH 4.2 is used.
