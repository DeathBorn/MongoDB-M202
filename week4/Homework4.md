Homework: 4.1: Limiting connections to a primary
----

Suppose you have a sharded cluster for which each shard is a 3-node replica set running MongoDB 2.6. You are concerned about limiting the number of connections in order to help ensure predictable behavior. Given the constraints of your system, you have decided to use 10GB as the amount of memory allocated to determine the maximum connections on the primary.

Using the formula discussed earlier in this chapter, determine the value to which you should set maxConns for each mongos in this cluster if you want to limit primary connections to a ~90% utilization level. Your cluster is using 64 mongos routers, and the number of other possible connections to a primary is 6.

Please calculate the maxConns value to achieve 90% utilization and round down to the nearest integer. Enter only the integer in the box below or your answer will be considered incorrect.

Note: For consistency with the lesson video, please assume that 10,000 connections consume 10GB of RAM.

Answer:

(maxPrimaryConnections - (numSecondaries x 3) - (numOther x 3)) / numMongos
(10000 - (2*3)  (6*3) )/64 =154,5625 
154,5625*0,9=139,10625 ~ 139

10000 
9000
64
6

var maxCons=10000
maxCons*0.9 = 9000
((maxCons*0.9)-6)/64
140.53125 ~140

Seen that on site there was a mistake on formula
SO WTF


Homework: 4.2: Optimizing a secondary for special case reads
----

Suppose you have an application on which you want to run analytics monthly. The analytics require an index and for performance reasons you will create the index on a secondary. Initiate a replica set with a primary and only one secondary. Create an index on the secondary only. The index should be on the "a" field of the "testDB.testColl" collection.

When you have created the index on the secondary, test with MongoProc to be sure you've completed the problem correctly and then submit.

_Note: If you have any documents in the testDB.testColl collection when you test or submit with MongoProc they will be removed._


Answer:

We can use the same database as in Homework 3.1. Just remote arbiter from replica set with 

```javascript
> rs.remove("127.0.0.1:30003")
```

And make sure that MongoProc connects to primary on the right port. Then start secondary 

```bash
$ mongod --dbpath mongod-sec

$ mongo
> use testDB
> db.testColl.ensureIndex({a:1})
{
	"createdCollectionAutomatically" : false,
	"numIndexesBefore" : 1,
	"numIndexesAfter" : 2,
	"ok" : 1
}
> db.testColl.stats()
{
	"ns" : "testDB.testColl",
	"count" : 100000,
	"size" : 24000000,
	"avgObjSize" : 240,
	"storageSize" : 37797888,
	"numExtents" : 8,
	"nindexes" : 2,
	"lastExtentSize" : 15290368,
	"paddingFactor" : 1,
	"systemFlags" : 1,
	"userFlags" : 1,
	"totalIndexSize" : 4235168,
	"indexSizes" : {
		"_id_" : 2518208,
		"a_1" : 1716960
	},
	"ok" : 1
}
> use admin
switched to db admin
> db.shutdownServer()

$ mongod --port 30001 --dbpath mongod-pri --replSet CorruptionTest --smallfiles --oplogSize 128 --config mongod-pri.conf
$ mongod --port 30002 --dbpath mongod-sec --replSet CorruptionTest --smallfiles --oplogSize 128 --config mongod-sec.conf

$ mongo --port 30002
> rs.stepDown()
```

Homework: 4.3: Triggering Rollback
----

In this problem, you will be causing a rollback scenario. Set up a replica set with one primary, one secondary, and an arbiter. Write some data to the primary in such a way that it does not replicate on the secondary (perhaps by taking down the secondary) then remove the primary from the replica set. Let the secondary come back up and become primary, write to it, and then bring up the original primary (it will now be a secondary).

Once you've triggered the rollback scenario, please submit your work using MongoProc. You do not need to reload the data lost during rollback. For your solution to be graded correct, you must have one secondary in the replica set on which rollback has occurred.

You must run mongoProc from the machine that experiences rollback. MongnoProc will look for the server at host='localhost' regardless of your settings, and you will need to direct it to the port that your primary is on.

Adam wrote a tutorial blog post on simulating rollback. It might be a little outdated, but provides a detailed discussion of this topic and is worth the read.


Answer:

```bash
mongo --nodb

var rst = new ReplSetTest({name:'testSet', nodes:{node1:{smallfiles:"",oplogSize:40, noprealloc:null}, node2:{smallfiles:"", oplogSize:40, noprealloc:null}, arb:{smallfiles:"",oplogSize:40,noprealloc:null}}});
rst.startSet()

mongo --port 31000
> rst.initiate()
> rs.status()
{
	"set" : "testSet",
	"date" : ISODate("2014-08-08T07:03:57Z"),
	"myState" : 1,
	"members" : [
		{
			"_id" : 0,
			"name" : "m2020:31001",
			"health" : 1,
			"state" : 1,
			"stateStr" : "PRIMARY",
			"uptime" : 88,
			"optime" : Timestamp(1407481403, 1),
			"optimeDate" : ISODate("2014-08-08T07:03:23Z"),
			"electionTime" : Timestamp(1407481404, 1),
			"electionDate" : ISODate("2014-08-08T07:03:24Z"),
			"self" : true
		}
	],
	"ok" : 1
}
> rs.addArb("m2020:31002")
> rs.add("m2020:31001")
> rs.status()
{
	"set" : "ad",
	"date" : ISODate("2014-08-08T07:11:22Z"),
	"myState" : 1,
	"members" : [
		{
			"_id" : 0,
			"name" : "m2020:31000",
			"health" : 1,
			"state" : 1,
			"stateStr" : "PRIMARY",
			"uptime" : 173,
			"optime" : Timestamp(1407481853, 1),
			"optimeDate" : ISODate("2014-08-08T07:10:53Z"),
			"electionTime" : Timestamp(1407481779, 1),
			"electionDate" : ISODate("2014-08-08T07:09:39Z"),
			"self" : true
		},
		{
			"_id" : 1,
			"name" : "m2020:31002",
			"health" : 1,
			"state" : 7,
			"stateStr" : "ARBITER",
			"uptime" : 56,
			"lastHeartbeat" : ISODate("2014-08-08T07:11:20Z"),
			"lastHeartbeatRecv" : ISODate("2014-08-08T07:11:21Z"),
			"pingMs" : 0
		},
		{
			"_id" : 2,
			"name" : "m2020:31001",
			"health" : 1,
			"state" : 2,
			"stateStr" : "SECONDARY",
			"uptime" : 29,
			"optime" : Timestamp(1407481853, 1),
			"optimeDate" : ISODate("2014-08-08T07:10:53Z"),
			"lastHeartbeat" : ISODate("2014-08-08T07:11:22Z"),
			"lastHeartbeatRecv" : ISODate("2014-08-08T07:11:22Z"),
			"pingMs" : 2,
			"syncingTo" : "m2020:31000"
		}
	],
	"ok" : 1
}


for(var i=0;i<100000;i++){db.test.insert({d:1})}


$ ps a | grep mongo
 8133 pts/1    Sl+    0:00 mongo --nodb
 8136 pts/1    Sl+    0:13 mongod --oplogSize 40 --port 31000 --noprealloc --smallfiles --rest --replSet ad --dbpath /data/db/ad-0 --setParameter enableTestCommands=1
 8211 pts/1    Sl+    0:13 mongod --oplogSize 40 --port 31001 --noprealloc --smallfiles --rest --replSet ad --dbpath /data/db/ad-1 --setParameter enableTestCommands=1
 8285 pts/1    Rl+    0:12 mongod --oplogSize 40 --port 31002 --noprealloc --smallfiles --rest --replSet ad --dbpath /data/db/ad-2 --setParameter enableTestCommands=1
 8361 pts/2    Sl+    0:00 mongo --port 31000
 8791 pts/3    S+     0:00 grep --color=auto mongo

$ kill -SIGSTOP 10151


> rs.status()
{
	"set" : "ad",
	"date" : ISODate("2014-08-08T07:19:22Z"),
	"myState" : 1,
	"members" : [
		{
			"_id" : 0,
			"name" : "m2020:31000",
			"health" : 1,
			"state" : 1,
			"stateStr" : "PRIMARY",
			"uptime" : 653,
			"optime" : Timestamp(1407482313, 844),
			"optimeDate" : ISODate("2014-08-08T07:18:33Z"),
			"electionTime" : Timestamp(1407481779, 1),
			"electionDate" : ISODate("2014-08-08T07:09:39Z"),
			"self" : true
		},
		{
			"_id" : 1,
			"name" : "m2020:31002",
			"health" : 1,
			"state" : 7,
			"stateStr" : "ARBITER",
			"uptime" : 536,
			"lastHeartbeat" : ISODate("2014-08-08T07:19:21Z"),
			"lastHeartbeatRecv" : ISODate("2014-08-08T07:19:20Z"),
			"pingMs" : 1
		},
		{
			"_id" : 2,
			"name" : "m2020:31001",
			"health" : 0,
			"state" : 8,
			"stateStr" : "(not reachable/healthy)",
			"uptime" : 0,
			"optime" : Timestamp(1407482250, 486),
			"optimeDate" : ISODate("2014-08-08T07:17:30Z"),
			"lastHeartbeat" : ISODate("2014-08-08T07:19:20Z"),
			"lastHeartbeatRecv" : ISODate("2014-08-08T07:17:30Z"),
			"pingMs" : 0,
			"syncingTo" : "m2020:31000"
		}
	],
	"ok" : 1
}


$kill -SIGINT 10108; kill -SIGCONT 10151

> rs.status()
{
	"set" : "ad",
	"date" : ISODate("2014-08-08T07:20:55Z"),
	"myState" : 1,
	"members" : [
		{
			"_id" : 0,
			"name" : "m2020:31000",
			"health" : 0,
			"state" : 8,
			"stateStr" : "(not reachable/healthy)",
			"uptime" : 0,
			"optime" : Timestamp(1407482250, 554),
			"optimeDate" : ISODate("2014-08-08T07:17:30Z"),
			"lastHeartbeat" : ISODate("2014-08-08T07:20:54Z"),
			"lastHeartbeatRecv" : ISODate("2014-08-08T07:20:15Z"),
			"pingMs" : 0
		},
		{
			"_id" : 1,
			"name" : "m2020:31002",
			"health" : 1,
			"state" : 7,
			"stateStr" : "ARBITER",
			"uptime" : 599,
			"lastHeartbeat" : ISODate("2014-08-08T07:20:53Z"),
			"lastHeartbeatRecv" : ISODate("2014-08-08T07:20:53Z"),
			"pingMs" : 1
		},
		{
			"_id" : 2,
			"name" : "m2020:31001",
			"health" : 1,
			"state" : 1,
			"stateStr" : "PRIMARY",
			"uptime" : 740,
			"optime" : Timestamp(1407482251, 73),
			"optimeDate" : ISODate("2014-08-08T07:17:31Z"),
			"electionTime" : Timestamp(1407482419, 1),
			"electionDate" : ISODate("2014-08-08T07:20:19Z"),
			"self" : true
		}
	],
	"ok" : 1
}


$ mongod --oplogSize 40 --port 31000 --smallfiles --rest --replSet testSet --dbpath /data/db/testSet-0 --setParameter enableTestCommands=1


> rs.status()
{
	"set" : "ad",
	"date" : ISODate("2014-08-08T07:22:29Z"),
	"myState" : 1,
	"members" : [
		{
			"_id" : 0,
			"name" : "m2020:31000",
			"health" : 1,
			"state" : 9,
			"stateStr" : "ROLLBACK",
			"uptime" : 12,
			"optime" : Timestamp(1407482313, 844),
			"optimeDate" : ISODate("2014-08-08T07:18:33Z"),
			"lastHeartbeat" : ISODate("2014-08-08T07:22:25Z"),
			"lastHeartbeatRecv" : ISODate("2014-08-08T07:22:27Z"),
			"pingMs" : 8,
			"lastHeartbeatMessage" : "replSet rollback 3 fixup",
			"syncingTo" : "m2020:31001"
		},
		{
			"_id" : 1,
			"name" : "m2020:31002",
			"health" : 1,
			"state" : 7,
			"stateStr" : "ARBITER",
			"uptime" : 693,
			"lastHeartbeat" : ISODate("2014-08-08T07:22:27Z"),
			"lastHeartbeatRecv" : ISODate("2014-08-08T07:22:27Z"),
			"pingMs" : 0
		},
		{
			"_id" : 2,
			"name" : "m2020:31001",
			"health" : 1,
			"state" : 1,
			"stateStr" : "PRIMARY",
			"uptime" : 834,
			"optime" : Timestamp(1407482251, 73),
			"optimeDate" : ISODate("2014-08-08T07:17:31Z"),
			"electionTime" : Timestamp(1407482419, 1),
			"electionDate" : ISODate("2014-08-08T07:20:19Z"),
			"self" : true
		}
	],
	"ok" : 1
}

> rs.status()
{
	"set" : "ad",
	"date" : ISODate("2014-08-08T07:22:48Z"),
	"myState" : 1,
	"members" : [
		{
			"_id" : 0,
			"name" : "m2020:31000",
			"health" : 1,
			"state" : 2,
			"stateStr" : "SECONDARY",
			"uptime" : 7,
			"optime" : Timestamp(1407482251, 73),
			"optimeDate" : ISODate("2014-08-08T07:17:31Z"),
			"lastHeartbeat" : ISODate("2014-08-08T07:22:47Z"),
			"lastHeartbeatRecv" : ISODate("2014-08-08T07:22:47Z"),
			"pingMs" : 1143,
			"lastHeartbeatMessage" : "syncing to: m2020:31001",
			"syncingTo" : "m2020:31001"
		},
		{
			"_id" : 1,
			"name" : "m2020:31002",
			"health" : 1,
			"state" : 7,
			"stateStr" : "ARBITER",
			"uptime" : 712,
			"lastHeartbeat" : ISODate("2014-08-08T07:22:47Z"),
			"lastHeartbeatRecv" : ISODate("2014-08-08T07:22:48Z"),
			"pingMs" : 0
		},
		{
			"_id" : 2,
			"name" : "m2020:31001",
			"health" : 1,
			"state" : 1,
			"stateStr" : "PRIMARY",
			"uptime" : 853,
			"optime" : Timestamp(1407482251, 73),
			"optimeDate" : ISODate("2014-08-08T07:17:31Z"),
			"electionTime" : Timestamp(1407482419, 1),
			"electionDate" : ISODate("2014-08-08T07:20:19Z"),
			"self" : true
		}
	],
	"ok" : 1
}

>rs.stepDown()
>rs.status()
{
	"set" : "ad",
	"date" : ISODate("2014-08-08T07:27:10Z"),
	"myState" : 2,
	"members" : [
		{
			"_id" : 0,
			"name" : "m2020:31000",
			"health" : 1,
			"state" : 1,
			"stateStr" : "PRIMARY",
			"uptime" : 269,
			"optime" : Timestamp(1407482739, 1),
			"optimeDate" : ISODate("2014-08-08T07:25:39Z"),
			"lastHeartbeat" : ISODate("2014-08-08T07:27:10Z"),
			"lastHeartbeatRecv" : ISODate("2014-08-08T07:27:10Z"),
			"pingMs" : 4,
			"electionTime" : Timestamp(1407482826, 1),
			"electionDate" : ISODate("2014-08-08T07:27:06Z")
		},
		{
			"_id" : 1,
			"name" : "m2020:31002",
			"health" : 1,
			"state" : 7,
			"stateStr" : "ARBITER",
			"uptime" : 974,
			"lastHeartbeat" : ISODate("2014-08-08T07:27:10Z"),
			"lastHeartbeatRecv" : ISODate("2014-08-08T07:27:10Z"),
			"pingMs" : 1
		},
		{
			"_id" : 2,
			"name" : "m2020:31001",
			"health" : 1,
			"state" : 2,
			"stateStr" : "SECONDARY",
			"uptime" : 1115,
			"optime" : Timestamp(1407482739, 1),
			"optimeDate" : ISODate("2014-08-08T07:25:39Z"),
			"self" : true
		}
	],
	"ok" : 1
}
```

Then use MongoProc on port 31000
