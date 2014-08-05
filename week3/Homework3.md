Homework 3.1: broken node, corrupt data files
----

In this problem we have provided you with a replica set that is corrupted. You will start out with a working primary, a corrupted secondary, and a working arbiter. Your goal is to recover the replica set by doing the necessary work to ensure the secondary has an uncorrupted version of the data.

Please download and extract the handouts. There are three data directories: one for the primary, secondary, and arbiter. On your host, you will need to run mongods for each of these on ports 30001, 30002, and 30003, respectively. **If you initially find that the mongod running on port 30002 is a primary, please step it down**. To get the replica set up and running, each node should be launched as follows:

1.  Primary:

```bash
mongod --port 30001 --dbpath mongod-pri --replSet CorruptionTest --smallfiles --oplogSize 128
```

2.  Secondary

```bash
mongod --port 30002 --dbpath mongod-sec --replSet CorruptionTest --smallfiles --oplogSize 128
```

3.  Arbiter

```bash
mongod --port 30003 --dbpath mongod-arb --replSet CorruptionTest
```

For this problem, do not attempt to configure MongoProc's port settings. You may still configure the host setting.

The corrupt data is in the data files found within the mongod-sec directory. Specifically, the testColl collection of the testDB database is corrupted. You can generate an error due to the corruption by running a .find().explain() query on this collection.

Bring the secondary back on line with uncorrupted data. When you believe you have done this, use MongoProc to test and, if correct, submit the homework

```bash
wget https://university.mongodb.com/static/10gen_2014_M202_July/handouts/chapter_3_disaster_recovery_and_backup.d7f194a0c43b.zip
unzip chapter_3_disaster_recovery_and_backup.d7f194a0c43b.zip
mv 3_1_broken_node_corrupt_data_files 3_1
cd 3_1
tar -xvf mongod-arb.tar.gz
tar -xvf mongod-pri.tar.gz
tar -xvf mongod-sec.tar.gz
rm -rf *.tar.gz


mongod --port 30001 --dbpath mongod-pri --replSet CorruptionTest --smallfiles --oplogSize 128 --config mongod-pri.conf
mongod --port 30002 --dbpath mongod-sec --replSet CorruptionTest --smallfiles --oplogSize 128 --config mongod-sec.conf
mongod --port 30003 --dbpath mongod-arb --replSet CorruptionTest --config mongod-arb.conf

mongo --port 30002

rs.stepDown()
2014-08-05T10:38:53.701+0000 DBClientCursor::init call() failed
2014-08-05T10:38:53.701+0000 Error: error doing query: failed at src/mongo/shell/query.js:81
2014-08-05T10:38:54.034+0000 trying reconnect to 127.0.0.1:30002 (127.0.0.1) failed
2014-08-05T10:38:54.041+0000 reconnect 127.0.0.1:30002 (127.0.0.1) ok

rs.status()
{
	"set" : "CorruptionTest",
	"date" : ISODate("2014-08-05T10:39:07Z"),
	"myState" : 2,
	"syncingTo" : "127.0.0.1:30001",
	"members" : [
		{
			"_id" : 0,
			"name" : "127.0.0.1:30001",
			"health" : 1,
			"state" : 1,
			"stateStr" : "PRIMARY",
			"uptime" : 119,
			"optime" : Timestamp(1399499925, 7077),
			"optimeDate" : ISODate("2014-05-07T21:58:45Z"),
			"lastHeartbeat" : ISODate("2014-08-05T10:39:06Z"),
			"lastHeartbeatRecv" : ISODate("2014-08-05T10:39:05Z"),
			"pingMs" : 0,
			"electionTime" : Timestamp(1407235135, 1),
			"electionDate" : ISODate("2014-08-05T10:38:55Z")
		},
		{
			"_id" : 1,
			"name" : "127.0.0.1:30002",
			"health" : 1,
			"state" : 2,
			"stateStr" : "SECONDARY",
			"uptime" : 124,
			"optime" : Timestamp(1399499925, 7077),
			"optimeDate" : ISODate("2014-05-07T21:58:45Z"),
			"infoMessage" : "syncing to: 127.0.0.1:30001",
			"self" : true
		},
		{
			"_id" : 2,
			"name" : "127.0.0.1:30003",
			"health" : 1,
			"state" : 7,
			"stateStr" : "ARBITER",
			"uptime" : 115,
			"lastHeartbeat" : ISODate("2014-08-05T10:39:07Z"),
			"lastHeartbeatRecv" : ISODate("2014-08-05T10:39:06Z"),
			"pingMs" : 0
		}
	],
	"ok" : 1
}

exit
mongo --port 30001
use testDB
db.testColl.find().explain()
{
	"cursor" : "BasicCursor",
	"isMultiKey" : false,
	"n" : 100000,
	"nscannedObjects" : 100000,
	"nscanned" : 100000,
	"nscannedObjectsAllPlans" : 100000,
	"nscannedAllPlans" : 100000,
	"scanAndOrder" : false,
	"indexOnly" : false,
	"nYields" : 912,
	"nChunkSkips" : 0,
	"millis" : 3893,
	"server" : "m2020:30001",
	"filterSet" : false
}
exit
```

Because secondary is corrupted and if it is ok to do initial sync [source](http://docs.mongodb.org/manual/tutorial/resync-replica-set-member/#automatically-sync-a-member) then just delete secondary dbpath and start mongod process

```bash
mongo --port 30002
use admin
db.shutdownServer()
exit
cd mongod-sec/
rm -rf journal local.*
rm -rf testDB.*
rm -rf mongod*
cd ..
mongod --port 30002 --dbpath mongod-sec --replSet CorruptionTest --smallfiles --oplogSize 128 --config mongod-sec.conf

mongo --port 30001
rs.status()
```

Homework 3.2
----

Operating at reduced capacity

You are running a service with a predictable traffic usage pattern. Every Monday the usage peaks at 100,000 reads/sec on your replica set at 17:00 UTC.

For each of the next four days, the peak is 2% higher than the previous day (so 102,000 reads/sec on Tuesday, 104,040 on Wednesday, 106,121 on Thursday, 108,243 on Friday) at 17:00 UTC each day. Saturday and Sunday see significantly reduced traffic, at peaks of 50,000. This pattern repeats each week.

You have nine servers that are evenly distributed across three data centers. The application uses a read preference of secondary. Each of the nine servers has been benchmarked to handle 24,000 reads/sec at its maximum capacity.

There is a further mandate to avoid exceeding 90% of available capacity, if at all possible, for performance reasons. If the application exceeds 90%, this must be reported and escalated to the executive level.

You have deployed the servers as follows:

-  Data Center A: Primary, Secondary, Secondary
-  Data Center B: Secondary, Secondary, Secondary
-  Data Center C: Secondary, Secondary, Secondary

A failure occurs at 19:00 UTC on a Saturday when Data Center C becomes unavailable, along with its servers. Based on the description above, by when must you fix the issue in order to be sure you do not exceed the 90% capacity rule and cause escalations to happen?

Assumptions:

-  The load is evenly distributed across all available secondaries, and redistributes itself shortlyafter the failure.
-  The load does not deviate from the prediction.
-  You cannot read from the primary.
-  All assumptions here are reasonable. ;-)

1.  By Sunday at 17:00 UTC
2.  By Monday at 17:00 UTC
3.  By Tuesday at 17:00 UTC
4.  By Wednesday at 17:00 UTC
5.  By Thursday at 17:00 UTC
6.  **By Friday at 17:00 UTC**
7.  By the next Saturday at 17:00 UTC

Explanations:

Peaks:

1.  Monday 100,000
2.  Tuesday 102,000
3.  Wednesday 104,040
4.  Thursday 106,121
5.  Friday 108,243
6.  Saturday 50,000
7.  Sunday 50,000

before failure read capacity is 8 * 24,000 = 192,000
and 90% of this is 172,800

if we loose datacenter DC3 then we loose three secondaries
and 5 * 24,000 = 120,000 and 90% of this is 108,000

So before friday we must fix it because   108,243 > 108,000

Homework 3.3: Network partition
----

Suppose you have a 3 member replica set with the primary in Data Center 2 (DC2) and two secondaries in Data Center 1 (DC1). You have set write concern at w=1. 

![backflush](/week3/data_center.jpg)

Now suppose that your application makes three writes to the primary. However, before these writes have been replicated to either of the secondaries, there is a network partition that prevents communication between DC2 and DC1. The partition lasts for 10 minutes. The application producing writes is able to see all three members of the replica set during the entire network partition between DC2 and DC1. No other problems with your system arise during this period. 

Which of the following are true about the system during the period of the network partition?

1.  **The application will still be able to read data.**
2.  The three writes to the primary before the network partition will be replicated to the secondaries when the partition ends.
3.  **The primary in DC2 will step itself down when the partition occurs.**
4.  An election will occur when the partition ends.
5.  **The two secondaries will hold an election when the partition occurs.**
6.  **The application will still be able to write while the partition is up.**
7.  Only reads will be possible while the partition is up.

Explanations:

w=1 write concern (Provides acknowledgment of write operations on a standalone mongod or the primary in a replica set. This is the default write concern for MongoDB. [source](http://docs.mongodb.org/manual/reference/write-concern/#w-option) ) 

1.  You allways can read if you have connection to database
2.  A rollback is necessary only if the primary had accepted write operations that the secondaries had not successfully replicated before the primary stepped down. When the primary rejoins the set as a secondary, it reverts, or “rolls back,” its write operations to maintain database consistency with the other members. [source](http://docs.mongodb.org/manual/core/replica-set-rollbacks/#rollbacks-during-replica-set-failover)
3.  A primary will step down if primary cannot contact a majority of the members of the replica set. [source](http://docs.mongodb.org/manual/core/replica-set-elections/#election-triggering-events)
4.  not true -> look into 3 and 5 answers.
5.  Replica sets hold an election any time there is no primary - a secondary loses contact with a primary. Secondaries call for elections when they cannot see a primary. [source](http://docs.mongodb.org/manual/core/replica-set-elections/#election-triggering-events)
6.  See 7 answer
7.  Network partitions affect the formation of a majority for an election. If a primary steps down and neither portion of the replica set has a majority the set will not elect a new primary. The replica set becomes read-only. So, in this situation there happens an election and application is able to see those members, so when we will have a primary then we will do writes. [source](http://docs.mongodb.org/manual/core/replica-set-elections/#network-partitions)

Homework 3.4: Using oplog replay to restore
----

Suppose that last night at midnight you took a snapshot of your database with mongodump. At a later time, someone accidentally dropped the only collection in the database using db.collection.drop(). You wish to restore the database to the state it was in immediately before the collection was dropped by replaying the oplog.

Which of the following are steps you should take in order for this to be successful? Check all that apply.
1.  **Prevent any writes that occurred after the drop() command from being replayed.**
2.  Prevent the writes that occurred before the backup from taking place.
3.  **Prevent the db.collection.drop() command in the oplog from being replayed.**

Explanations:

1.  From the task itsef '__o restore the database to the state it was in immediately before the collection was dropped__'
2.  You want everything before drop
3.  You don't want that collection to be dropped again


Homework 3.5:Replaying the oplog
----

This problem will be a hands-on implementation of the last problem.

The backupDB database has one collection, backupColl. At midnight every night, the system is backed up with a mongodump. Your server continued taking writes for a few hours, until 02:46:39. At that point, someone (not you) ran the command:

```javascript
> db.backupColl.drop()
```
Your job is to put your database back into the state it was in immediately before the database was dropped, then use MongoProc to verify that you have done it correctly.
You have been provided with a server directory in the backuptest.tar.gz file that includes your (now empty) data files, the mongodump file in backupDB.tar.gz, and a mongod.conf file. The conf file will set the name of your replica set to "BackupTest" and the port to 30001. Your replica set must have these settings to be graded correct in MongoProc. You may configure the host setting for MongoProc if necessary.

Use MongoProc to evaluate your solution. You can verify that your database is in the correct state with the test button and turn it in once you've passed.

This assignment is fairly tricky so you may want to check this stackoverflow question and answer.

__Tip: You may not need this, but if you are interested in updating an oplog's 'op' field for a document, it will complain if you increase the size the document (which it thinks is happening as an intermediate stage of an update), but you can do it anyway by simultaneously unsetting another field. For example, db.oplog.rs.update( { query }, { $set : { "op" : "c" }, $unset : { "o" : 1 } } )__.

```bash
$ mv 3_5_replaying_the_oplog 3_5
$ cd 3_5
$ tar -xvf backupDB.tar.gz
$ tar -xvf backuptest.tar.gz
$ mongod --dbpath backuptest --config mongod.conf
$ mongo --port 30001
>use local
>db.oplog.rs.find({op:'c'})
{ "ts" : Timestamp(1398778745, 1), "h" : NumberLong("-4262957146204779874"), "v" : 2, "op" : "c", "ns" : "backupDB.$cmd", "o" : { "drop" : "backupColl" } }

$ mongodump -d local -c oplog.rs -o oplogD --dbpath backuptest
2014-08-05T16:09:31.463+0000 DATABASE: local	 to 	oplogD/local
2014-08-05T16:09:31.469+0000 [tools] info openExisting file size 16777216 but storageGlobalParams.smallfiles=false: backuptest/local.0
2014-08-05T16:09:31.510+0000 	local.oplog.rs to oplogD/local/oplog.rs.bson
2014-08-05T16:09:31.635+0000 [tools] getmore local.oplog.rs cursorid:14960592282 ntoreturn:0 keyUpdates:0 numYields:0 locks(micros) r:121125 nreturned:25116 reslen:4194392 121ms
2014-08-05T16:09:31.729+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:09:31.862+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:09:31.978+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:09:32.101+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:09:32.197+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:09:32.326+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:09:32.471+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:09:32.623+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:09:32.746+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:09:32.872+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:09:32.997+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:09:33.114+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:09:33.232+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:09:33.377+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:09:33.482+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:09:33.514+0000 		 701202 documents
2014-08-05T16:09:33.517+0000 	Metadata for local.oplog.rs to oplogD/local/oplog.rs.metadata.json
2014-08-05T16:09:33.522+0000 [tools] dbexit:
2014-08-05T16:09:33.522+0000 [tools] shutdown: going to close listening sockets...
2014-08-05T16:09:33.522+0000 [tools] shutdown: going to flush diaglog...
2014-08-05T16:09:33.522+0000 [tools] shutdown: going to close sockets...
2014-08-05T16:09:33.522+0000 [tools] shutdown: waiting for fs preallocator...
2014-08-05T16:09:33.522+0000 [tools] shutdown: closing all files...
2014-08-05T16:09:33.550+0000 [tools] closeAllFiles() finished
2014-08-05T16:09:33.551+0000 [tools] shutdown: removing fs lock...
2014-08-05T16:09:33.551+0000 [tools] dbexit: really exiting now

$ cd backuptest
$ rm -rf *
$ cd ..
$ mongorestore --dbpath backuptest backupDB
2014-08-05T16:10:40.013+0000 backupDB/backupColl.bson
2014-08-05T16:10:40.015+0000 	going into namespace [backupDB.backupColl]
2014-08-05T16:10:40.019+0000 [FileAllocator] allocating new datafile backuptest/backupDB.ns, filling with zeroes...
2014-08-05T16:10:40.020+0000 [FileAllocator] creating directory backuptest/_tmp
2014-08-05T16:10:40.058+0000 [FileAllocator] done allocating datafile backuptest/backupDB.ns, size: 16MB,  took 0.003 secs
2014-08-05T16:10:40.074+0000 [FileAllocator] allocating new datafile backuptest/backupDB.0, filling with zeroes...
2014-08-05T16:10:40.077+0000 [FileAllocator] done allocating datafile backuptest/backupDB.0, size: 64MB,  took 0.001 secs
2014-08-05T16:10:40.091+0000 [tools] build index on: backupDB.backupColl properties: { v: 1, key: { _id: 1 }, name: "_id_", ns: "backupDB.backupColl" }
2014-08-05T16:10:40.093+0000 [tools] 	 added index to empty collection
2014-08-05T16:10:43.005+0000 [tools] 		Progress: 3712800/55036800	6%	(bytes)
2014-08-05T16:10:44.270+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:10:46.003+0000 [tools] 		Progress: 7516600/55036800	13%	(bytes)
2014-08-05T16:10:48.443+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:10:49.007+0000 [tools] 		Progress: 11329500/55036800	20%	(bytes)
2014-08-05T16:10:52.007+0000 [tools] 		Progress: 15124200/55036800	27%	(bytes)
2014-08-05T16:10:52.682+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:10:55.008+0000 [tools] 		Progress: 18873400/55036800	34%	(bytes)
2014-08-05T16:10:56.945+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:10:58.055+0000 [tools] 		Progress: 22549800/55036800	40%	(bytes)
2014-08-05T16:11:01.006+0000 [tools] 		Progress: 26044200/55036800	47%	(bytes)
2014-08-05T16:11:01.474+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:11:01.672+0000 [FileAllocator] allocating new datafile backuptest/backupDB.1, filling with zeroes...
2014-08-05T16:11:01.675+0000 [FileAllocator] done allocating datafile backuptest/backupDB.1, size: 128MB,  took 0.001 secs
2014-08-05T16:11:04.003+0000 [tools] 		Progress: 29747900/55036800	54%	(bytes)
2014-08-05T16:11:05.755+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:11:07.004+0000 [tools] 		Progress: 33451600/55036800	60%	(bytes)
2014-08-05T16:11:10.006+0000 [tools] 		Progress: 36918700/55036800	67%	(bytes)
2014-08-05T16:11:10.364+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:11:13.003+0000 [tools] 		Progress: 40467700/55036800	73%	(bytes)
2014-08-05T16:11:14.740+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:11:16.013+0000 [tools] 		Progress: 44235100/55036800	80%	(bytes)
2014-08-05T16:11:19.009+0000 [tools] 		Progress: 47793200/55036800	86%	(bytes)
2014-08-05T16:11:19.140+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:11:22.011+0000 [tools] 		Progress: 51287600/55036800	93%	(bytes)
2014-08-05T16:11:23.751+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:11:25.008+0000 [tools] 		Progress: 54736500/55036800	99%	(bytes)
604800 objects found
2014-08-05T16:11:25.256+0000 	Creating index: { key: { _id: 1 }, name: "_id_", ns: "backupDB.backupColl" }
2014-08-05T16:11:25.258+0000 [tools] dbexit:
2014-08-05T16:11:25.259+0000 [tools] shutdown: going to close listening sockets...
2014-08-05T16:11:25.259+0000 [tools] shutdown: going to flush diaglog...
2014-08-05T16:11:25.259+0000 [tools] shutdown: going to close sockets...
2014-08-05T16:11:25.259+0000 [tools] shutdown: waiting for fs preallocator...
2014-08-05T16:11:25.259+0000 [tools] shutdown: closing all files...
2014-08-05T16:11:25.269+0000 [tools] closeAllFiles() finished
2014-08-05T16:11:25.270+0000 [tools] shutdown: removing fs lock...
2014-08-05T16:11:25.271+0000 [tools] dbexit: really exiting now

$ mkdir oplogR
$ cp oplogD/local/oplog.rs.bson oplogR/oplog.bson
$ mongorestore --dbpath backuptest --oplogReplay --oplogLimit 1398778745:1 oplogR
2014-08-05T16:12:18.038+0000 	 Replaying oplog
2014-08-05T16:12:21.009+0000 [tools] 		Progress: 3924419/113990182	3%	(bytes)
2014-08-05T16:12:24.004+0000 [tools] 		Progress: 7915719/113990182	6%	(bytes)
2014-08-05T16:12:25.381+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:12:27.007+0000 [tools] 		Progress: 11790119/113990182	10%	(bytes)
2014-08-05T16:12:27.323+0000 [tools] command backupDB.$cmd command: applyOps { applyOps: [ { ts: Timestamp 1398771852000|27225, h: 3563893781373949669, v: 2, op: "i", ns: "backupDB.backupColl", o: { _id: new Date(1388605624000), string: "testStringForPadding0000000000000000000000000000000000000000" } } ] } ntoreturn:1 keyUpdates:0 numYields:0 locks(micros) W:218040 reslen:68 218ms
2014-08-05T16:12:30.006+0000 [tools] 		Progress: 14495519/113990182	12%	(bytes)
2014-08-05T16:12:33.010+0000 [tools] 		Progress: 17818819/113990182	15%	(bytes)
2014-08-05T16:12:34.640+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:12:36.011+0000 [tools] 		Progress: 20808119/113990182	18%	(bytes)
2014-08-05T16:12:39.033+0000 [tools] 		Progress: 23396619/113990182	20%	(bytes)
2014-08-05T16:12:42.007+0000 [tools] 		Progress: 25834819/113990182	22%	(bytes)
2014-08-05T16:12:45.036+0000 [tools] 		Progress: 28907619/113990182	25%	(bytes)
2014-08-05T16:12:45.407+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:12:48.068+0000 [tools] 		Progress: 30627719/113990182	26%	(bytes)
2014-08-05T16:12:51.014+0000 [tools] 		Progress: 32965719/113990182	28%	(bytes)
2014-08-05T16:12:54.013+0000 [tools] 		Progress: 36756619/113990182	32%	(bytes)
2014-08-05T16:12:55.860+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:12:57.016+0000 [tools] 		Progress: 40464019/113990182	35%	(bytes)
2014-08-05T16:13:00.017+0000 [tools] 		Progress: 44121319/113990182	38%	(bytes)
2014-08-05T16:13:02.626+0000 [tools] command backupDB.$cmd command: applyOps { applyOps: [ { ts: Timestamp 1398771860000|5113, h: -3342875448337168795, v: 2, op: "i", ns: "backupDB.backupColl", o: { _id: new Date(1388812792000), string: "testStringForPadding0000000000000000000000000000000000000000" } } ] } ntoreturn:1 keyUpdates:0 numYields:0 locks(micros) W:119095 reslen:68 119ms
2014-08-05T16:13:03.018+0000 [tools] 		Progress: 46743219/113990182	41%	(bytes)
2014-08-05T16:13:05.427+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:13:06.007+0000 [tools] 		Progress: 49665719/113990182	43%	(bytes)
2014-08-05T16:13:09.008+0000 [tools] 		Progress: 53640319/113990182	47%	(bytes)
2014-08-05T16:13:12.012+0000 [tools] 		Progress: 57030419/113990182	50%	(bytes)
2014-08-05T16:13:13.231+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:13:15.018+0000 [tools] 		Progress: 60971619/113990182	53%	(bytes)
2014-08-05T16:13:18.006+0000 [tools] 		Progress: 64912819/113990182	56%	(bytes)
2014-08-05T16:13:20.985+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:13:21.006+0000 [tools] 		Progress: 68469919/113990182	60%	(bytes)
2014-08-05T16:13:24.015+0000 [tools] 		Progress: 72260819/113990182	63%	(bytes)
2014-08-05T16:13:27.012+0000 [tools] 		Progress: 75467219/113990182	66%	(bytes)
2014-08-05T16:13:29.548+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:13:30.010+0000 [tools] 		Progress: 78857319/113990182	69%	(bytes)
2014-08-05T16:13:33.009+0000 [tools] 		Progress: 82831919/113990182	72%	(bytes)
2014-08-05T16:13:36.037+0000 [tools] 		Progress: 86672919/113990182	76%	(bytes)
2014-08-05T16:13:37.162+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:13:39.003+0000 [tools] 		Progress: 90330219/113990182	79%	(bytes)
2014-08-05T16:13:42.005+0000 [tools] 		Progress: 94020919/113990182	82%	(bytes)
2014-08-05T16:13:44.861+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:13:45.012+0000 [tools] 		Progress: 97995519/113990182	85%	(bytes)
2014-08-05T16:13:48.013+0000 [tools] 		Progress: 101761355/113990182	89%	(bytes)
2014-08-05T16:13:51.010+0000 [tools] 		Progress: 104643355/113990182	91%	(bytes)
2014-08-05T16:13:52.580+0000 [tools] warning Listener::getElapsedTimeMillis returning 0ms
2014-08-05T16:13:54.007+0000 [tools] 		Progress: 107525355/113990182	94%	(bytes)
2014-08-05T16:13:57.009+0000 [tools] 		Progress: 110394255/113990182	96%	(bytes)
2014-08-05T16:14:00.010+0000 [tools] 		Progress: 113555719/113990182	99%	(bytes)
701202 objects found
2014-08-05T16:14:00.365+0000 Applied 701200 oplog entries out of 701201 (1 skipped).
2014-08-05T16:14:00.366+0000 [tools] dbexit:
2014-08-05T16:14:00.367+0000 [tools] shutdown: going to close listening sockets...
2014-08-05T16:14:00.367+0000 [tools] shutdown: going to flush diaglog...
2014-08-05T16:14:00.368+0000 [tools] shutdown: going to close sockets...
2014-08-05T16:14:00.370+0000 [tools] shutdown: waiting for fs preallocator...
2014-08-05T16:14:00.371+0000 [tools] shutdown: closing all files...
2014-08-05T16:14:00.379+0000 [tools] closeAllFiles() finished
2014-08-05T16:14:00.381+0000 [tools] shutdown: removing fs lock...
2014-08-05T16:14:00.382+0000 [tools] dbexit: really exiting now
