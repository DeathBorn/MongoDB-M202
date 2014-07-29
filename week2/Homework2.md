Homework 2.1
----
Homework 2.2
----

You are operating a geographically dispersed replica set as outlined below:

![backflush](/week2/replica_set_chaining.png)

Your network operations team has asked if you can limit the amount of outbound traffic from Site A because of some capacity issues with the traffic in and out of that site. For example, consider that this is where the majority of your users stream/download. However, due to how the application is deployed, you wish to keep your write traffic in Site A for performance, and so you have chosen not to change which server is primary.

Using replica set chaining, which of the following scenarios will minimize traffic due to MongoDB replication in and out of Site A?

Note: It will probably help you to draw each of the choices below for yourself as you think through them.

1.  P to S1; P to S2; P to S3; P to S4 (All secondaries replicate from the primary.)
2.  **P to S1; S1 to S2; S2 to S3; S3 to S4**
3.  P to S2; S2 to S1; S2 to S3; S3 to S4
4.  P to S4; S4 to S1; S4 to S2; S4 to S3
5.  P to S1; S1 to S2; S2 to S3; S1 to S4



Homework 2.3
----

You are performing an aggregation query with $sort, and are hitting the maximum size limit for in-memory sort. Which of the following might resolve this problem?


1.  **If running MongoDB 2.4, move your system to another machine with more memory**
2.  Switch out your HDD for an SSD so that data can be accessed more quickly
3.  Move your system to another machine with a faster CPU
4.  **Add an index for the variable(s) you are using to sort the documents**
5.  **If you are not already doing so, include a $match earlier in the pipeline that will reduce the number of documents you are sorting**

Explanations:

The $sort stage has a limit of 100 megabytes of RAM. By default, if the stage exceeds this limit, $sort will produce an error. [source](http://docs.mongodb.org/manual/reference/operator/aggregation/sort/#sort-and-memory-restrictions)

1.  Changed in version 2.6: The memory limit for $sort changed from 10 percent of RAM to 100 megabytes of RAM. [source](http://docs.mongodb.org/manual/reference/operator/aggregation/sort/#sort-and-memory-restrictions)
2.  HDD and SSD does not increase the memory limit
3.  faster CPU does not increase the memory limit
4.  "$sort operator can take advantage of an index when placed at the beginning of the pipeline or placed before the following aggregation operators: $project, $unwind, and $group." [source](http://docs.mongodb.org/manual/reference/operator/aggregation/sort/#sort-and-memory-restrictions)
5.  "Place the $match as early in the aggregation pipeline as possible. Because $match limits the total number of documents in the aggregation pipeline, earlier $match operations minimize the amount of processing down the pipe. If you place a $match at the very beginning of a pipeline, the query can take advantage of indexes like any other db.collection.find() or db.collection.findOne()." [source](http://docs.mongodb.org/manual/reference/operator/aggregation/match/#pipeline-optimization) 




Homework 2.4
----

Note: MongoProc validation for this exercise requires that you are running MongoDB 2.6

Over time a MongoDB database can become fragmented and cause reduced storage efficiency.

In this problem we have provided you with an already fragmented database. You will want to extract the handout and set your mongod's dbpath to the extracted 'fragDB' directory. You'll be working with the fragDB database, and the fragColl collection.

Your goal in this homework is to increase storage efficiency and reclaim disk space. You may use any method learned thus far to accomplish this goal. However, please do not tamper (remove/update/insert) with the documents in the database given, apart from querying the database to find out more information about the fragmented components.

You are encouraged to use MongoProc to continuously test your progress until you are ready to grade the homework.

Useful Formulas

Collection Storage Efficiency:
```javascript
db.fragColl.stats().size/db.fragColl.stats().storageSize
```
Database Storage Efficiency:
```javascript
(db.stats().dataSize + db.stats().indexSize) / db.stats().fileSize
```


Answers:

```bash
mongod --dbpath /tmp/fragDB
```

```log
MongoDB starting : pid=3109 port=27017 dbpath=/tmp/fragDB 64-bit host=m2020
2014-07-29T12:35:50.690+0000 [initandlisten] db version v2.6.3
2014-07-29T12:35:50.691+0000 [initandlisten] git version: 255f67a66f9603c59380b2a389e386910bbb52cb
2014-07-29T12:35:50.692+0000 [initandlisten] build info: Linux build12.nj1.10gen.cc 2.6.32-431.3.1.el6.x86_64 #1 SMP Fri Jan 3 21:39:27 UTC 2014 x86_64 BOOST_LIB_VERSION=1_49
2014-07-29T12:35:50.692+0000 [initandlisten] allocator: tcmalloc
2014-07-29T12:35:50.693+0000 [initandlisten] options: { storage: { dbPath: "/tmp/fragDB" } }
2014-07-29T12:36:14.172+0000 [initandlisten] journal dir=/tmp/fragDB/journal
2014-07-29T12:36:14.180+0000 [initandlisten] recover : no journal files present, no recovery needed
2014-07-29T12:36:47.956+0000 [initandlisten] preallocateIsFaster=true 674.46
2014-07-29T12:36:50.681+0000 [initandlisten] preallocateIsFaster=true 53.54
2014-07-29T12:36:53.923+0000 [initandlisten] preallocateIsFaster=true 42.9
2014-07-29T12:36:53.923+0000 [initandlisten] preallocateIsFaster check took 39.742 secs
2014-07-29T12:36:53.923+0000 [initandlisten] preallocating a journal file /tmp/fragDB/journal/prealloc.0
2014-07-29T12:36:56.023+0000 [initandlisten] 		File Preallocator Progress: 597688320/1073741824	55%
2014-07-29T12:37:02.563+0000 [initandlisten] preallocating a journal file /tmp/fragDB/journal/prealloc.1
2014-07-29T12:37:11.344+0000 [initandlisten] preallocating a journal file /tmp/fragDB/journal/prealloc.2
2014-07-29T12:37:38.859+0000 [initandlisten] query admin.system.roles planSummary: EOF ntoreturn:0 ntoskip:0 nscanned:0 nscannedObjects:0 keyUpdates:0 numYields:0 locks(micros) W:2038 r:629591 nreturned:0 reslen:20 629ms
2014-07-29T12:37:39.209+0000 [FileAllocator] allocating new datafile /tmp/fragDB/local.ns, filling with zeroes...
2014-07-29T12:37:39.210+0000 [FileAllocator] creating directory /tmp/fragDB/_tmp
2014-07-29T12:37:39.689+0000 [FileAllocator] done allocating datafile /tmp/fragDB/local.ns, size: 16MB,  took 0.1 secs
2014-07-29T12:37:40.020+0000 [FileAllocator] allocating new datafile /tmp/fragDB/local.0, filling with zeroes...
2014-07-29T12:37:46.788+0000 [FileAllocator] done allocating datafile /tmp/fragDB/local.0, size: 64MB,  took 6.766 secs
2014-07-29T12:37:46.790+0000 [initandlisten] ExtentManager took 6 seconds to open: /tmp/fragDB/local.0
2014-07-29T12:37:47.334+0000 [initandlisten] build index on: local.startup_log properties: { v: 1, key: { _id: 1 }, name: "_id_", ns: "local.startup_log" }
2014-07-29T12:37:47.463+0000 [initandlisten] 	 added index to empty collection
2014-07-29T12:37:47.964+0000 [initandlisten] command local.$cmd command: create { create: "startup_log", size: 10485760, capped: true } ntoreturn:1 keyUpdates:0 numYields:0  reslen:37 8856ms
2014-07-29T12:37:48.635+0000 [initandlisten] insert local.startup_log ninserted:1 keyUpdates:0 numYields:0  668ms
2014-07-29T12:37:48.804+0000 [initandlisten] waiting for connections on port 27017
```

```bash
$mongo
```

```javascript
> use fragDB
> db.fragColl.stats().size/db.fragColl.stats().storageSize
0.14289938663772606
> (db.stats().dataSize + db.stats().indexSize) / db.stats().fileSize
0.12821715218680246
> exit
```

```bash
$ mongod --dbpath /tmp/fragDB --repair
```

```log
2014-07-29T12:43:47.659+0000 [initandlisten] MongoDB starting : pid=3795 port=27017 dbpath=/tmp/fragDB 64-bit host=m2020
2014-07-29T12:43:47.661+0000 [initandlisten] db version v2.6.3
2014-07-29T12:43:47.661+0000 [initandlisten] git version: 255f67a66f9603c59380b2a389e386910bbb52cb
2014-07-29T12:43:47.661+0000 [initandlisten] build info: Linux build12.nj1.10gen.cc 2.6.32-431.3.1.el6.x86_64 #1 SMP Fri Jan 3 21:39:27 UTC 2014 x86_64 BOOST_LIB_VERSION=1_49
2014-07-29T12:43:47.662+0000 [initandlisten] allocator: tcmalloc
2014-07-29T12:43:47.662+0000 [initandlisten] options: { repair: true, storage: { dbPath: "/tmp/fragDB" } }
2014-07-29T12:43:47.703+0000 [initandlisten] repairDatabase fragDB
2014-07-29T12:43:47.705+0000 [FileAllocator] allocating new datafile /tmp/fragDB/_tmp_repairDatabase_0/fragDB.ns, filling with zeroes...
2014-07-29T12:43:47.706+0000 [FileAllocator] creating directory /tmp/fragDB/_tmp_repairDatabase_0/_tmp
2014-07-29T12:43:47.709+0000 [FileAllocator] done allocating datafile /tmp/fragDB/_tmp_repairDatabase_0/fragDB.ns, size: 16MB,  took 0.001 secs
2014-07-29T12:43:47.719+0000 [FileAllocator] allocating new datafile /tmp/fragDB/_tmp_repairDatabase_0/fragDB.0, filling with zeroes...
2014-07-29T12:43:47.722+0000 [FileAllocator] done allocating datafile /tmp/fragDB/_tmp_repairDatabase_0/fragDB.0, size: 64MB,  took 0.001 secs
2014-07-29T12:43:49.023+0000 [initandlisten] warning Listener::getElapsedTimeMillis returning 0ms
2014-07-29T12:43:50.310+0000 [initandlisten] warning Listener::getElapsedTimeMillis returning 0ms
2014-07-29T12:43:51.867+0000 [initandlisten] warning Listener::getElapsedTimeMillis returning 0ms
2014-07-29T12:43:53.782+0000 [initandlisten] warning Listener::getElapsedTimeMillis returning 0ms
2014-07-29T12:43:55.346+0000 [initandlisten] warning Listener::getElapsedTimeMillis returning 0ms
2014-07-29T12:43:56.386+0000 [initandlisten] warning Listener::getElapsedTimeMillis returning 0ms
2014-07-29T12:43:57.819+0000 [initandlisten] warning Listener::getElapsedTimeMillis returning 0ms
2014-07-29T12:43:59.654+0000 [initandlisten] warning Listener::getElapsedTimeMillis returning 0ms
2014-07-29T12:44:01.539+0000 [initandlisten] warning Listener::getElapsedTimeMillis returning 0ms
2014-07-29T12:44:02.749+0000 [initandlisten] warning Listener::getElapsedTimeMillis returning 0ms
2014-07-29T12:44:03.766+0000 [initandlisten] warning Listener::getElapsedTimeMillis returning 0ms
2014-07-29T12:44:04.836+0000 [initandlisten] warning Listener::getElapsedTimeMillis returning 0ms
2014-07-29T12:44:06.102+0000 [initandlisten] warning Listener::getElapsedTimeMillis returning 0ms
2014-07-29T12:44:08.130+0000 [initandlisten] warning Listener::getElapsedTimeMillis returning 0ms
2014-07-29T12:44:09.638+0000 [initandlisten] warning Listener::getElapsedTimeMillis returning 0ms
2014-07-29T12:44:11.251+0000 [initandlisten] warning Listener::getElapsedTimeMillis returning 0ms
2014-07-29T12:44:12.681+0000 [initandlisten] warning Listener::getElapsedTimeMillis returning 0ms
2014-07-29T12:44:14.032+0000 [initandlisten] warning Listener::getElapsedTimeMillis returning 0ms
2014-07-29T12:44:15.566+0000 [initandlisten] warning Listener::getElapsedTimeMillis returning 0ms
2014-07-29T12:44:19.015+0000 [initandlisten] warning Listener::getElapsedTimeMillis returning 0ms
2014-07-29T12:44:22.378+0000 [initandlisten] warning Listener::getElapsedTimeMillis returning 0ms
2014-07-29T12:44:29.873+0000 [initandlisten] warning Listener::getElapsedTimeMillis returning 0ms
2014-07-29T12:44:38.418+0000 [initandlisten] warning Listener::getElapsedTimeMillis returning 0ms
2014-07-29T12:44:43.864+0000 [initandlisten] warning Listener::getElapsedTimeMillis returning 0ms
2014-07-29T12:44:47.565+0000 [initandlisten] warning Listener::getElapsedTimeMillis returning 0ms
2014-07-29T12:45:15.563+0000 [DataFileSync] flushing mmaps took 27901ms  for 8 files
2014-07-29T12:45:16.683+0000 [initandlisten] warning Listener::getElapsedTimeMillis returning 0ms
2014-07-29T12:45:21.095+0000 [initandlisten] warning Listener::getElapsedTimeMillis returning 0ms
2014-07-29T12:45:24.446+0000 [initandlisten] warning Listener::getElapsedTimeMillis returning 0ms
2014-07-29T12:45:29.436+0000 [initandlisten] warning Listener::getElapsedTimeMillis returning 0ms
2014-07-29T12:45:32.451+0000 [FileAllocator] allocating new datafile /tmp/fragDB/_tmp_repairDatabase_0/fragDB.1, filling with zeroes...
2014-07-29T12:45:32.454+0000 [FileAllocator] done allocating datafile /tmp/fragDB/_tmp_repairDatabase_0/fragDB.1, size: 128MB,  took 0.001 secs
2014-07-29T12:45:46.278+0000 [initandlisten] repairDatabase local
2014-07-29T12:45:46.278+0000 [FileAllocator] allocating new datafile /tmp/fragDB/_tmp_repairDatabase_0/local.ns, filling with zeroes...
2014-07-29T12:45:46.278+0000 [FileAllocator] creating directory /tmp/fragDB/_tmp_repairDatabase_0/_tmp
2014-07-29T12:45:46.287+0000 [FileAllocator] done allocating datafile /tmp/fragDB/_tmp_repairDatabase_0/local.ns, size: 16MB,  took 0 secs
2014-07-29T12:45:46.294+0000 [FileAllocator] allocating new datafile /tmp/fragDB/_tmp_repairDatabase_0/local.0, filling with zeroes...
2014-07-29T12:45:46.297+0000 [FileAllocator] done allocating datafile /tmp/fragDB/_tmp_repairDatabase_0/local.0, size: 64MB,  took 0.001 secs
2014-07-29T12:45:46.728+0000 [initandlisten] finished checking dbs
2014-07-29T12:45:46.728+0000 [initandlisten] dbexit:
2014-07-29T12:45:46.729+0000 [initandlisten] shutdown: going to close listening sockets...
2014-07-29T12:45:46.731+0000 [initandlisten] shutdown: going to flush diaglog...
2014-07-29T12:45:46.732+0000 [initandlisten] shutdown: going to close sockets...
2014-07-29T12:45:46.733+0000 [initandlisten] shutdown: waiting for fs preallocator...
2014-07-29T12:45:46.735+0000 [initandlisten] shutdown: closing all files...
2014-07-29T12:45:46.735+0000 [initandlisten] closeAllFiles() finished
2014-07-29T12:45:46.737+0000 [initandlisten] shutdown: removing fs lock...
2014-07-29T12:45:46.737+0000 [initandlisten] dbexit: really exiting now
```

```bash
$mongo
```

```javascript
>use fragDB
> db.fragColl.stats()
{
	"ns" : "fragDB.fragColl",
	"count" : 200000,
	"size" : 48000000,
	"avgObjSize" : 240,
	"storageSize" : 58441728,
	"numExtents" : 9,
	"nindexes" : 1,
	"lastExtentSize" : 20643840,
	"paddingFactor" : 1,
	"systemFlags" : 0,
	"userFlags" : 1,
	"totalIndexSize" : 5044592,
	"indexSizes" : {
		"_id_" : 5044592
	},
	"ok" : 1
}
> db.fragColl.stats().size/db.fragColl.stats().storageSize
0.8213309503784693
> (db.stats().dataSize + db.stats().indexSize) / db.stats().fileSize
0.2634766101837158
```

Alternatively you can run db.repairDatabase() in mongo shell

