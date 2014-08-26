Homework: 6.1: Using shard tags to manage data
----

In this problem we will emulate a data management strategy in which we periodically move data from short-term storage (STS) to long-term storage (LTS). We have implemented this strategy using tag-based sharding.

Start by spinning up a sharded cluster as follows:

```bash
$ mongo --nodb
> config = { d0 : { smallfiles : "", noprealloc : "", nopreallocj : ""}, d1 : { smallfiles : "", noprealloc : "", nopreallocj : "" }, d2 : { smallfiles : "", noprealloc : "", nopreallocj : ""}};
> cluster = new ShardingTest( { shards : config } );
```

This will create 3 standalone shards on ports 30000, 30001, and 30002, as well as a mongos on port 30999. The config portion of the above will eliminate the disk space issues some students have seen when using ShardingTest.

Next, initialize the data in this cluster using MongoProc. You can choose the host you're pointing to, but MongoProc will be looking for the mongos at port 30999.

Following initialization, your system will contain the collection testDB.testColl. Once initial balancing completes, all documents in this collection with a createdDate field value that falls any time in the year 2013 are in LTS and all documents created in 2014 are in STS. There are two shards used for LTS and one shard for STS.

Your assignment is to move all data for the month of January 2014 into LTS as part of periodic maintenance. For this problem we are pretty sure you can "solve" it in a couple of ways that are not ideal. In an ideal solution you will make the balancer do the work for you. Please note that the balancer must be running when you turn in your solution.



Explanations:

```javascript
mongo --nodb
config = { d0 : { smallfiles : "", noprealloc : "", nopreallocj : ""}, d1 : { smallfiles : "", noprealloc : "", nopreallocj : "" }, d2 : { smallfiles : "", noprealloc : "", nopreallocj : ""}};
cluster = new ShardingTest( { shards : config } );

#initialize from MongoProc
sh.stopBalancer()

db.tags.remove({ "tag": "STS" });
db.tags.remove({ "tag": "LTS" });

sh.addTagRange("testDB.testColl", { "createdDate" : ISODate("2014-02-01T00:00:00Z") }, { "createdDate" : ISODate("2014-05-01T00:00:00Z") }, "STS");
sh.addTagRange("testDB.testColl", { "createdDate" : ISODate("2013-10-01T00:00:00Z") }, { "createdDate" : ISODate("2014-02-01T00:00:00Z") }, "LTS")

sh.startBalancer()
```


Homework: 6.2: Traffic Imbalance in a Sharded Environment
----


In this problem, you have a cluster with 2 shards, each with a similar volume of data, but all the application traffic is going to one shard. You must diagnose the query pattern that is causing this problem and figure out how to balance out the traffic.

To set up the scenario, run the following commands to set up your cluster. The config document passed to ShardingTest will eliminate the disk space issues some students have seen when using ShardingTest.

```bash
$ mongo --nodb
> config = { d0 : { smallfiles : "", noprealloc : "", nopreallocj : ""}, d1 : { smallfiles : "", noprealloc : "", nopreallocj : "" } };
> cluster = new ShardingTest( { shards: config } );
```

Once the cluster is up, click "Initialize" in MongoProc one time to finish setting up the cluster's data and configuration. If you are running MongoProc on a machine other than the one running the mongos, then you must change the host of 'mongod1' in the settings. The host should be the hostname or IP of the machine on which the mongos is running. MongoProc will use port 30999 to connect to the mongos for this problem.

Once the cluster is initialized, click the "Initialize" button in MongoProc again to simulate application traffic to the cluster for 1 minute. You may click "Initialize" as many times as you like to simulate more traffic for 1 minute at a time. If you need to begin the problem again and want MongoProc to reinitialize the dataset, drop the week6 database from the cluster and click "Initialize" in MongoProc.

Use diagnostic tools (e.g., mongostat and the database profiler) to determine why all application traffic is being routed to one shard. Once you believe you have fixed the problem and traffic is balanced evenly between the two shards, test using MongoProc and then turn in if the test completes successfully.


Explanations:

```javascript
sh.stopBalancer()
sh.addShardTag("shard0000","left");
sh.addShardTag("shard0001","right");
sh.addTagRange("week6.m202", { "_id" : MinKey }, { "_id" : ISODate("2014-07-16T00:00:00Z") }, "left");
sh.addTagRange("week6.m202", { "_id" : ISODate("2014-07-16T00:00:00Z") }, { "_id" : MaxKey }, "right");
sh.startBalancer()

```

This answer is not elegant way. Maybe, I should have dumped data and created collection with "otherID" shard key.

Homework: 6.3: Finding configuration errors in log files
----

Download, extract, and examine the log files in the attachment. These are the log files for four servers that were (by the end of the logs) spun up into a single replica set.

Initially, there was a problem with one or more server configuration options. In the log file(s), find the configuration setting that is initially causing the issue.

Enter the field name (not the value) for the setting that is the source of the problem. Do not use numbers, quotation marks, or other punctuation in your answer. For example, if you thought the problem was "logAppend": true (which it definitely isn't), you would type:

logAppend

When you have your answer, please paste or type it into the box below, and submit. Unfortunately, case/capitalization matters here, so please enter the field name for the setting in question exactly how it appears in the logs.

Answer: "bindIp"

Explanations:

We have four server

mba:27017 host

```javascript
config: "m202-sec.conf", 
net: { bindIp: "127.0.0.1", port: 27017 }, 
processManagement: { fork: true }, 
replication: { oplogSizeMB: 512, replSetName: "m202-final" }, 
storage: { dbPath: "/data/db/rs2" }, 
systemLog: { destination: "file", logAppend: true, path: "/data/db/m202-mba.log" } 
```

mbp:27017 host

```javascript
pid=89449 port=27017 dbpath=/data/db/rs2 64-bit host=mbp.10gen.cc
config: "m202-sec.conf", 
net: { port: 27017 }, 
processManagement: { fork: true }, 
replication: { oplogSizeMB: 512, replSetName: "m202-final" }, 
storage: { dbPath: "/data/db/rs2" }, 
systemLog: { destination: "file", logAppend: true, path: "/data/db/m202-mbp.log" } 
```

m202-ubuntu:27017 host

```javascript
config: "./m202-pri.conf", 
net: { bindIp: "127.0.0.1", port: 27017 }, 
processManagement: { fork: true }, 
replication: { oplogSizeMB: 512, replSetName: "m202-final" }, 
storage: { dbPath: "/data/db/rs1" }, 
systemLog: { destination: "file", logAppend: true, path: "/data/db/m202-ubuntu-27017.log" } 
```

m202-ubuntu:27018

```javascript
config: "./m202-sec.conf", 
net: { bindIp: "127.0.0.1", port: 27018 }, 
processManagement: { fork: true }, 
replication: { oplogSizeMB: 512, replSetName: "m202-final" }, 
storage: { dbPath: "/data/db/rs2" }, 
systemLog: { destination: "file", logAppend: true, path: "/data/db/m202-ubuntu-27018.log" } 
```

Here above you see initial configuration. Firstly m202-ubuntu:27018 and m202-ubuntu:27018 hosts became online and communicated (same host). Then others tried, however bindIp configuration option did not allowed to communicate to other hosts, so thats way they got "connection refused" exception. [source](http://docs.mongodb.org/manual/reference/configuration-options/#net.bindIp)
