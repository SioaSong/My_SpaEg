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

Installation
------------

_This page explains how to do a basic installation of the HBase Indexer on a single machine._<br>

Before you start, make sure that you have the required software installed (they can all be running on single machine).<br>
Get the HBase Indexer

Check out the code and build the tar.gz distribution.

git clone git://github.com/NGDATA/hbase-indexer.git
mvn clean package -Pdist -DskipTests
Next, unpackage the tar.gz distribution (in the example below it is unpacked under your $HOME directory).

```
tar zxvf hbase-indexer-dist/target/hbase-indexer-1.0-SNAPSHOT.tar.gz -C ~
cd ~/hbase-indexer-1.0-SNAPSHOT
Configure HBase Indexer
```

In the hbase-indexer directory, edit the file conf/hbase-indexer-site.xml and configure the ZooKeeper connection string (twice, once for hbase-indexer, and once for hbase, alternatively you can copy your hbase-site.xml to the conf directory).
> 
```
<property>
  <name>hbaseindexer.zookeeper.connectstring</name>
  <value>zookeeperhost</value>
</property>
<property>
  <name>hbase.zookeeper.quorum</name>
  <value>zookeeperhost</value>
</property>
```
If you have not defined JAVA_HOME globally, and the bin/hbase-indexer script would complain it doesn't find you Java, you can set the JAVA_HOME in the script conf/hbase-indexer-env.sh.

Configure HBase

In order to use the HBase Indexer, replication must be enabled in HBase. There are also a number of other HBase settings that can be set to optimize the working of the HBase indexer.

Add the settings below to your hbase-site.xml configuration on all HBase region servers, and restart HBase.

```
<configuration>
  <!-- SEP is basically replication, so enable it -->
  <property>
    <name>hbase.replication</name>
    <value>true</value>
  </property>
  <!-- Source ratio of 100% makes sure that each SEP consumer is actually
       used (otherwise, some can sit idle, especially with small clusters) -->
  <property>
    <name>replication.source.ratio</name>
    <value>1.0</value>
  </property>
  <!-- Maximum number of hlog entries to replicate in one go. If this is
       large, and a consumer takes a while to process the events, the
       HBase rpc call will time out. -->
  <property>
    <name>replication.source.nb.capacity</name>
    <value>1000</value>
  </property>
  <!-- A custom replication source that fixes a few things and adds
       some functionality (doesn't interfere with normal replication
       usage). -->
  <property>
    <name>replication.replicationsource.implementation</name>
    <value>com.ngdata.sep.impl.SepReplicationSource</value>
  </property>
</configuration>
```
__Add indexer jars to HBase__

The HBase Indexer includes two jar files that need to be in the classpath of HBase. Copy these from the lib directory of the unpacked hbase-indexer installation into the lib directory of HBase for each region server.
> 
```
cp lib/hbase-sep-* $HBASE_HOME/lib
```
__Start Solr__

Ensure that Solr is running. In general, it's easiest to have Solr use the same ZooKeeper instance as HBase.

Assuming that you've downloaded Solr 4.2.0 and you're running ZooKeeper on the current machine, you can start up the base Solr in cloud mode using the example schema as follows:
> 
```
cd $SOLR_HOME/example
java -Dbootstrap_confdir=./solr/collection1/conf -Dcollection.configName=myconf -DzkHost=localhost:2181/solr  -jar start.jar
```
### -DzkHost的值视solr定义而定，有可能会是localhost:2181，或者IP:端口号/solr 或者 IP:端口号 的形式。

Tutorial
--------

_This page explains how to start doing basic indexing in HBase. Before following this tutorial, make sure that the HBase Indexer and other required software is installed and running as explained in the installation instructions._

At this point, HBase and Solr (in cloud mode) should be running, and the HBase Indexer should be unpacked in a directory (which we'll call $INDEXER_HOME). For this tutorial, it is assumed that the default example index schema is being used by Solr (as explained on the installation page).

Start the HBase Indexer daemon

In a terminal, execute the following (assuming $INDEXER_HOME points to the directory where the hbase-indexer tar.gz distribution was unpacked).
> 
```
cd $INDEXER_HOME
./bin/hbase-indexer server
Create a table to be indexed in HBase
```

In the HBase shell, create a table. For this tutorial, we'll create a table named "indexdemo-user", with a single column family named "info". Note that the REPLICATION_SCOPE of the column family of the table must be set to 1.

```
$ hbase shell
hbase> create 'indexdemo-user', { NAME => 'info', REPLICATION_SCOPE => '1' }
Add an indexer
```

Now we'll create an indexer that will index the the indexdemo-user table as its contents are updated.

In your favourite text editor, create a new xml file called indexdemo-indexer.xml, with the following content:

```
<?xml version="1.0"?>
<indexer table="indexdemo-user">
  <field name="firstname_s" value="info:firstname"/>
  <field name="lastname_s" value="info:lastname"/>
  <field name="age_i" value="info:age" type="int"/>
</indexer>
```

The above file defines three pieces of information that will be used for indexing, how to interpret them, and how they will be stored in Solr.

Next, create an indexer based on the created indexer xml file.
```
./bin/hbase-indexer add-indexer -n myindexer -c indexdemo-indexer.xml \
        -cp solr.zk=localhost:2181/solr -cp solr.collection=collection1
```
__注意此处：solr.zk=localhost:2181/solr 与之前启动solr的配置有关，可在页面查看 solr的 zkHost参数值确定此处内容 __<br>
Note that the above command assumes that ZooKeeper is running on localhost on port 2181, and that there is a Solr Core called "collection1" configured. If you are doing this tutorial on an existing HBase/Solr environment, you may need to use different settings.

Update the table content

In the HBase shell, try adding some data to the indexdemo-user table

```
hbase> put 'indexdemo-user', 'row1', 'info:firstname', 'John'
hbase> put 'indexdemo-user', 'row1', 'info:lastname', 'Smith'
```
After adding this data, take a look in Solr (i.e. http://localhost:8983/solr/#/collection1/query). You should see a single document in Solr that has the firstname_s field set to "John", and the lastname_s field set to "Smith".

Note If you don't have autoCommit enabled in Solr, you won't be able to see the updated contents immediately in Solr. The demo environment has autoCommit enabled for a commit every second.

Now try updating the data you've just added
```
hbase> put 'indexdemo-user', 'row1', 'info:firstname', 'Jim'
```
And now check the content in Solr. The document's firstname_s field now contains the string "Jim".

Finally, delete the row from HBase.
> 
```
hbase> deleteall 'indexdemo-user', 'row1'
```
You can now verify that the data has been deleted from Solr.
> 
```
./bin/hbase-indexer add-indexer -n Testindexer -c /gfire/YsFiel/hbindexdemo-Yanis.xml -cp solr.zk=localhost:2181 -cp solr.collection=indexer_test

create 'indexdemoTest', { NAME => 'info', REPLICATION_SCOPE => '1' }

hbase> put 'indexdemoTest', 'row1', 'info:firstname', 'John'
hbase> put 'indexdemoTest', 'row1', 'info:lastname', 'Smith'

hbase> deleteall 'indexdemo-user', 'row1'
```
