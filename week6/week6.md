SHARDED CLUSTER MANAGEMENT, PART 2

Anatomy of migration ovierview

-why?
-misunderstood(blame) 
	- lightweight(maximum chunk size 64MB)
	- 1 at any time (cluster-wide)
-Touches all components in sharded cluster

db.changelog.find().sort()

focus on moveChunk.from and moveChunk.to

Caveat: Prone to Change!

steps
From            To
1                 1
2                 2
3                 3
4   goes to 1     4
5                 5 goes to 5 to left
6                



mongo --nodb
sh.enableSharding("shardTest")
sh.shardCollection("shardTest.foo",{"_id":1})
use shardTest
for(var i=0; i<=100000; i++){db.foo.insert({a:i})}
sh.status() 

what: "moveChunk.from"
==== what: "moveChunk.commit"
what: "moveChunk.to"
what: "moveChunk.start"




################

Anatomy of a migration deep dive

mongos      c c c

FROM TO

1. mongos -> balance round -> looks for lock (config has lock collection)
2. identify imbalance, pick a chunk to migrate
3. Send command to FROM shard (step 1-FROM)
4. FROM performs sanity checks (step 2-FROM) (can reject because wrong or too busy)
5. FROM sends command to TO shard (step 3-FROM)
6. Transfer Stars (step 4-FROM,Step 1-3-TO)
	check if it has index data, if not it will create indexes
	check if there is any documents exists there already
7. Catchup on Subsequent Ops (because mongos still does actions on FROm when migration) (step 4-FROM, Step 4-T)
8. Catchup finishd, Transfer Complete(Steady)(step6-T)
	everything is loged in changelog
9. Critical Section (about to commit) (Step5-FROM) (goes to config servers)(critical is because you can have inconsistent state in primaries)
10. Clean Up (Step6-F) (remove chunks which are moved)
11. Subsequent request to FROM shard for chunk get stale config exeption, trigger mongos refresh (mongos will refresh their metadata from config servers) (version counter)



While a chunk is in flight, where are updates and inserts for documents in the chunk range routed?
	They are not routed, the mongos caches them until the chunk migration completes.
	====They are routed to the shard the chunk is being migrated from.
	They are routed to the shard the chunk is being migrated to.
	They are routed to both shards.
	They are not routed, clients receive an exception notifying them to retry once migration is complete.

#########
System-level migration options

-waitForDelete (false=default)
	From side impact. it will not move on until clean up ops when migration. makes slower migration but makes less load on FROM primary
-secondaryThrottle (true=default)
	W:2  write concern is 2
	if you have small documents and many sometimes you want to set this false


################
Tag based Sharding

- "Pin" collections, or ranges, to specific shards

3 regions usa and emea and apac

S1U S2U S3A S4E S5E

so mongos use correct prefix


############
Simple tag-based sharding example

-Approrpiate shard key
	{id, user_id, other...
		region:"EMEA"....}
	{id, user_id, other...
		region:"US"....}

	Compound Shard Key
	so region and user id
	{region:1,user_id:1,other:1}
	so chunk ranges {USa25632 -> USb324552}

so 2 steps:
	1) designated tags to shards

	sh1 -USEAST
	sh2 -EMEA
	sh3 -APAC
	sh4 -USEAST

		sh.status()
		sh.addShardTag("test-rs0","USWEST")
		sh.addShardTag("test-rs1","USEAST")

		use config
		db.shards.find()

	2) need to tag ranges.
		{ $Minkey -> "APAC252524" }
		{ "APAC252524" ->}

		prepend letters and presplit
		USEAST
		EMEA
		APAC
		USEAST

		for region in [] {
			for(x=0;x<16;x++){
				for(y=0;y<16;y++){
					prefix = region + x+ y + padding;
					db.adminCommand("split")
				}
			}
		}
		it will give 1024 -> 4 regions, 256 chunks eachn

		sh.addTagRange("regiondata.users", {$minKey}, {"APAC....."}, "APAC")
		and add to other regions




mongo --nodb

config = { d0 : { smallfiles : "", noprealloc : "", nopreallocj : ""}, d1 : { smallfiles : "", noprealloc : "", nopreallocj : "" }, d2 : { smallfiles : "", noprealloc : "", nopreallocj : ""}};
cluster = new ShardingTest( { shards : config } );
// shard db
sh.enableSharding("houses");

// shard collections
sh.shardCollection("houses.stark", {dire_wolves_owned:1});
sh.shardCollection("houses.lannister", {debt_owed:1});
sh.shardCollection("houses.targaryen", {followers:1});

// Insert sample data
use houses;
var bulk = db.stark.initializeUnorderedBulkOp();

for (var i=0; i < 100000; i++) { bulk.insert({dire_wolves_owned: Math.random()}); }
bulk.execute();
bulk = db.lannister.initializeUnorderedBulkOp();
for (var i=0; i < 100000; i++) { bulk.insert({debt_owed: Math.random()}); }
bulk.execute();
bulk = db.targaryen.initializeUnorderedBulkOp();
for (var i=0; i < 100000; i++) { bulk.insert({followers: Math.random()}); }

bulk.execute();

sh.addShardTag("shard0000", "sta");
sh.addShardTag("shard0001", "lan");
sh.addShardTag("shard0002", "tar");

sh.addTagRange("houses.stark", {dire_wolves_owned:MinKey}, {dire_wolves_owned:MaxKey}, "sta");
sh.addTagRange("houses.lannister", {debt_owed:MinKey}, {debt_owed:MaxKey}, "lan");
sh.addTagRange("houses.targaryen", {followers:MinKey}, {followers:MaxKey}, "tar");



#################
Tag-based sharding caveats

data is messy, without presplit. several shards

zipcode 
	NY 100-200
	NJ 200-300

chunk {min 150-> max 220}

which will choose. it will choose by closest lower bound of chunk to other max limits of shards. shortest distance winds

balancer respects tags for future balancing

	   s1  s2  s3
chunks 32  31  32

you want pin collection for s1 and s2 shards only
do it, all tag ranges, then leave, come back. nothing happend. the reason is because it will respect for future. how to move these chunks from s3 to s2 or s1. do it MANUALY.

-manually move

sh.moveChunk(collection, chunk, shard)
sh.moveChunk("shardTEst.foo",{_id:ObjectId("dasfsdf")},"test-rs1")

you want to stop balancer if you move chunks or splits (only one migration at the time)
sh.stopBalancer()

###########
Hash Based sharding

-new in 2.4
-similar consideratiosn to pre-splitting
	-high cardinality (lots of values)
	-floating point numbers can be problematic (bad choice)
-pick a field and create an index wioth computetd hash
	ensureIndex command {_id:"hashed"}
-shardCollection command pass in {_id:"hashed"}
-only supports single values 

##########
demo

sh.enableSharding()

db.bar.ensureIndex("_id":"hashed")
sh.shardCollection("hashTest.bar",{"_id":"hashed"})

if you will have hash based on empty collection then by default it will create 2 chunks on each shard

db.adminCommand({shardCollection:"hashTest.bar",key:{"_id":1},numInitialChunks:2048})
if you want more chunks when createing


60GB, 4 shards, 10K average size, 6m documents = 1920

##############
Hash based sharding
-benefits
	-very good for scaling writes (if you really need that)
-drawbacks
	-1 document -> scatter gather, or any other ranged based query. mongos will have to wait for all answers from shards. so will be as slow as slowest shard
	no-specific queries -> all shards
		only as fast as slowest shard
	-single field only
	-not supported tag based shards, cant do it.

In which of the following scenarios would hash-based sharding be most likely to provide performance benefits?
	====In write-heavy applications
	In read-heavy applications (could be if locality would not be issue)
	In systems distributed across hardware with varying levels of performance
	If the common query patterns in an application select documents based on multiple fields


###############
Unbalanced Chunks
-Number of chunks -> only thing the balancer cares about
	does not considerr size (amount of data)
	does not consider number of documents
	crude balancing measurements
-splits(auto), balancer (ussually takes care of this problem)

Scenarios where imbalance occurs
2 main categories
1) lots of empty chunks
2) some number of large chunks


Consider the diagram below depicting the distribution of chunks across a sharded cluster. For each chunk, the number (16, 32, or 64) defines the size of the chunk in MB. Assuming no new chunks are created, which of the following best describes what actions the balancer will take during the next balancing round?

s1 s2 s3
32 16 64
   16
32 16
   16
64 16 64
   16
   16
   16
 =====
3  8  2

	The shards are balanced. There will not be a balancing round until more chunks are created.
	The balancer will migrate a chunk from shard 1 to shard 2.
	====The balancer will migrate a chunk from shard 2 to shard 3.
	The balancer will migrate a chunk from shard 3 to shard 1.
	The balancer will migrate a chunk from shard 1 to shard 3.

BECAUSE of the count. from the most to the least count.

###########
scenarios where imbalance occurs

empty chunk scenario:
-pre-splitting mistake 
-time based shard key (not good option)
	periodically deleting old data

s1           s2
500 chunks   newly added
on _id

pick a chunk? lowest range
balances 

250 newest   250 oldest

then delete 6 months data old
balancer thinks it is balanced bnut you have no data, all trafic goes to s1 only.
PIT FALL

what to do with empty chunks?
-stop balancer
-manually move chunks
- from 2.6+ mergeChunk
	only merge chunks that:
		are on the same shard
		contigoues ranges (next to each other)
		one , or both , must be empty


config = { d0 : { smallfiles : "", noprealloc : "", nopreallocj : ""}, d1 : { smallfiles : "", noprealloc : "", nopreallocj : "" }};
cluster = new ShardingTest( { shards : config } );
// shard db
sh.enableSharding("emptyChunks");

sh.shardCollection("emptyChunks.bar", {email:1});

for ( var x=97; x<97+26; x++ ) {
    for( var y=97; y<97+26; y+=6 ) {
        var prefix = String.fromCharCode(x) + String.fromCharCode(y);
        db.runCommand( { split : "emptyChunks.bar", middle : { email : prefix } } );
    }
}

#verify if empty

db.runCommand({
   "dataSize": "emptyChunks.bar",
   "keyPattern": { email: 1 },
   "min": { "email": "aa" },
   "max": { "email": "ag" }
})
{ "size" : 0, "numObjects" : 0, "millis" : 0, "ok" : 1 }

db.runCommand( { mergeChunks: "emptyChunks.bar",
                 bounds: [ { "email": "aa" },
                           { "email": "am" } ]
             } )


Answer

First thing you'll need to do is spin up a cluster for this test. You can pick your number of shards & replica set configuration, but here's a simple example with 3-node replica sets:

> cluster = new ShardingTest( { shards: 1, rs: { nodes: [ { } ] } } );
Once that has spun up, connect to the mongos:

$ mongo --port 30999
then enable sharding on your database and shard your collection:

mongos> sh.enableSharding("myapp")
{ "ok" : 1 }
mongos> sh.shardCollection( "myapp.users" , { "email" : 1 } )
{ "collectionsharded" : "myapp.users", "ok" : 1 }
mongos> 
Now, we can look at our chunks and see that initially we have only one:

mongos> use config
switched to db config
mongos> db.chunks.find()
{ "_id" : "myapp.users-email_MinKey", "lastmod" : Timestamp(1, 0), "lastmodEpoch" : ObjectId("538e27be31972172d9b3ec61"), "ns" : "myapp.users", "min" : { "email" : { "$minKey" : 1 } }, "max" : { "email" : { "$maxKey" : 1 } }, "shard" : "test-rs0" }
mongos>
Great! So let's now break that apart with the script in the problem statement:

mongos> use admin
switched to db admin
mongos> for ( var x=97; x<97+26; x++ ) {
...     for( var y=97; y<97+26; y+=6 ) {
...         var prefix = String.fromCharCode(x) + String.fromCharCode(y);
...         db.runCommand( { split : "myapp.users", middle : { email : prefix } } );
...     }
... }
{ "ok" : 1 }
mongos>
And now we can see that we've got lots of chunks:

mongos> use config
switched to db config
mongos> db.chunks.find().count()
131
So now to merge a few of them. First, we'll want to verify that they're empty. Of course, we know that these are empty (since we haven't put any data in them), but in production, things might not be so obvious. I'll set some variables to make querying easy, then verify the size of data in it:

mongos> first_doc = db.chunks.find().next()
{
	"_id" : "myapp.users-email_MinKey",
	"lastmod" : Timestamp(2, 0),
	"lastmodEpoch" : ObjectId("538e27be31972172d9b3ec61"),
	"ns" : "myapp.users",
	"min" : {
		"email" : { "$minKey" : 1 }
	},
	"max" : {
		"email" : "aa"
	},
	"shard" : "test-rs1"
}
mongos> min = first_doc.min
{ "email" : { "$minKey" : 1 } }
mongos> max = first_doc.max
{ "email" : "aa" }
mongos> keyPattern = { email : 1 }
{ "email" : 1 }
mongos> ns = first_doc.ns
myapp.users
mongos> db.runCommand({dataSize: ns, keyPattern: keyPattern, min: min, max: max } )
{ "size" : 0, "numObjects" : 0, "millis" : 0, "ok" : 1 }
So, that last line tells us that there are 0 documents in that first chunk. We could do this with any number of chunks, and of course the answer will be the same.

OK, let's merge the first and second docs, first finding the shard key maximum on the second doc:

mongos> second_doc = db.chunks.find().skip(1).next()
{
	"_id" : "myapp.users-email_\"aa\"",
	"lastmod" : Timestamp(3, 0),
	"lastmodEpoch" : ObjectId("538e27be31972172d9b3ec61"),
	"ns" : "myapp.users",
	"min" : {
		"email" : "aa"
	},
	"max" : {
		"email" : "ag"
	},
	"shard" : "test-rs0"
}
mongos> max2 = second_doc.max
{ "email" : "ag" }
mongos> use admin
switched to db admin
mongos> db.runCommand( { mergeChunks : ns , bounds : [ min , max2 ] } )
{ "ok" : 1 }
So there we've done it! Let's look at our chunk data:

mongos> use config
mongos> db.chunks.count()
130
mongos> db.chunks.find().limit(2)
{ "_id" : "myapp.users-email_MinKey", "ns" : "myapp.users", "min" : { "email" : { "$minKey" : 1 } }, "max" : { "email" : "ag" }, "version" : Timestamp(1, 261), "versionEpoch" : ObjectId("538e2cc240ba884cdd64c109"), "lastmod" : Timestamp(1, 261), "lastmodEpoch" : ObjectId("538e2cc240ba884cdd64c109"), "shard" : "test-rs0" }
{ "_id" : "myapp.users-email_\"ag\"", "lastmod" : Timestamp(1, 5), "lastmodEpoch" : ObjectId("538e2cc240ba884cdd64c109"), "ns" : "myapp.users", "min" : { "email" : "ag" }, "max" : { "email" : "am" }, "shard" : "test-rs0" }
So we've eliminated a chunk (count went from 131 to 130), and we can see that the first chunk's range goes all the way from $minKey to 'ag'.



##################
Data Imbalance Scenario

-poor shard key(became big chunks)
-pre-splitting error
-Auto-splits require traffic

what to do/
-wait for auto splits balancing
-manual splits -> wait for balancer
-manual splits and manual moves

AllChunkInfo script will be available

#############
Orphaned Chunks
- temporary orphaned chunks
-permanet orphaned chunks

during inflight of chunk data from home shard to target shard. some data 50% on target shard are orphaned chunk

        s

s1                 s2
chunk  inflight    

if s2 goes away  then replica set in s2 will find new primary. so in-flight  migration aborted but 50% percent transfer was replicated already. that data still is orphaned chunk data


STILL BUG!!!!!!!!!
orphaned documents 
	counted by the count command (will couse to count twise) space waste

orphaned documents can show up on secondary reads and if you connect straigt to shard

if migrate successfull then home chunk becomes orphaned and is deleting them in bacground (this is temporaly). Hovewer if primary goes away and delete did not begin in other members of the set same things will happen -permanent orphaned documents.

Do orphaned chunks alter the data returned for queries with a Primary read preference?
	====No, because they are filtered out by the chunk manager
	No, because orphaned documents are marked by a field MongoDB automatically adds
	No, because they are automatically deleted
	Yes, because they will be retrieved for non-targeted (scatter/gather) queries
	Yes, because all documents matching a query are included in the result set


#########################
How to deal with orphaned chunks?
pre 2.6
	was script - orphanage.js in support-tools
for 2.6+
	cleanupOrphaned command (secondaryThrottle -> write concern here)

#################
Removing a Shard
-should not be done lightly
-going to be slow (approach with caution)

-run the removeShard command
-wait for all chunks to be migrated off
-move any databases that have that shard as their primary (movePrimary -this moves databases)
-run the removeShard command again

mark as draining. balancer running must be run for that. that means that shard will not be a target. all chunks will move to ather shards. 

sh.startBalancer()
db.adminCommand({removeShard:"adf"})
will show which databases to move

db.shards.find()
draining true

#############
Examining Log Files

mongos, mongos -> log files

show log

logs can be very large -> compress well, because repetition in logs
logs can be cryptic
	lots of noise

####################
grep "socket exception" mongod.log
The output probably won't fit on one screen, but you only need to see the last line:
Apr  5 22:06:18 m202-ubuntu mongod.27017[17999]: Fri Apr  5 22:06:18
[rsHealthPoll] replSet info m202-ubuntu-3:27017 is down (or slow to respond):
socket exception [CONNECT_ERROR] for m202-ubuntu-3:27017
You could also open the file in vim or less, jump to the end with G, and search backward for "socket exception" with ?socket exception.
##################
Filtering out connectin from log files

how many lines
wc -l  mongod.log

grep accepted mongod.log | wc -l

grep 'end.connection' mongod.log | wc -l

-v does not have
grep -v end.conn mongod.log | grep -v accepted.from | w -l
# eliminating noise

grep exception filtered-mongod.log | wc -l

-i case insensitive
grep exception filtered-mongod.log | grep -iv socket | wc -l


Probably the first thing to do is glance at anything with the word "connection" in it.
grep connection mongod.log | less
Most of these lines resemble this:
Apr  5 06:39:57 m202-ubuntu mongod.27017[808]: Fri Apr  5 06:39:57
[initandlisten] connection accepted from 127.0.0.1:15991 #8952 (59 connections
now open)
We're only interested in the number of connections, so let's focus on the 59 connections now open part. There are many ways to do this, but one is to take advantage of the fact that . matches any character in grep:
$ grep '(. connections now open' mongod.log | wc -l
      19
$ grep '(.. connections now open' mongod.log | wc -l
   49548
$ grep '(... connections now open' mongod.log | wc -l
       0
$ grep '(.... connections now open' mongod.log | wc -l
       0
So most of the time, the number of open connections was a 2-digit number, and it was never 3 or 4 digits. 100 connections is pretty reasonable, so the performance problem is probably being caused by something else.

#############################
grepping log files

you want to get all slowest ops and sort by first longest

grep -v oplog mongod.log | grep -o "[0-9]\+ms" | sort -n | tail

grep 1006369ms mongod.log     (about 25 minutes)

majority always must be satisfied

grep -v writeback <filename> | grep -v oplog | grep -o "[0-9]\+ms" | sort -n | tail -n 20 | uniq | xargs -n 1 -J x grep x <filename>

looking only for milisecond value, sort limiting, unique

grep -v writeback <filename> | grep -v oplog | grep -o "[0-9]\+ms" | sort -n | tail -n 20 | uniq | xargs -n 1 -I x grep x <filename>

egrep "[0-9]{3,}ms" mongod.lot | awk '{ print $NF, $0 }' | sort -n | tail -n 5
match 3 or more digits, print the last element in space separated line, $0 current line


The following analysis finds one such type of query. You can start with a pipeline that finds the top 5 slowest operations:
egrep '[0-9]{3,}ms$' mongod.log | awk '{print $NF, $0}' | sort -n | tail -n 5
Let's ignore all the getLastError operations. Let's also use -i so we don't have to think about whether it's spelled getlasterror or getLastError:
egrep '[0-9]{3,}ms$' mongod.log | grep -iv getlasterror | awk '{print $NF, $0}' | sort -n | tail -n 5
16872ms Apr  5 16:00:26 m202-ubuntu mongod.27017[808]: Fri Apr  5 16:00:26
[conn16030] query serverside.scrum_master query: { $query: { datetime_used: {
$ne: null } }, $orderby: { _id: 1 } } cursorid:6213124304229200070 ntoreturn:10
ntoskip:0 nscanned:7000376 keyUpdates:0 numYields: 51490 locks(micros)
r:32180103 nreturned:10 reslen:2060 16872ms

16965ms Apr  5 15:57:45 m202-ubuntu mongod.27017[808]: Fri Apr  5 15:57:45
[conn16030] query serverside.scrum_master query: { $query: { datetime_used: {
$ne: null } }, $orderby: { _id: 1 } } cursorid:8343223522342664909 ntoreturn:10
ntoskip:0 nscanned:7000376 keyUpdates:0 numYields: 48058 locks(micros)
r:32188994 nreturned:10 reslen:2060 16965ms

17017ms Apr  5 16:30:35 m202-ubuntu mongod.27017[808]: Fri Apr  5 16:30:35
[conn16030] query serverside.scrum_master query: { $query: { datetime_used: {
$ne: null } }, $orderby: { _id: 1 } } cursorid:3380969414085642540 ntoreturn:10
ntoskip:0 nscanned:7000376 keyUpdates:0 numYields: 52433 locks(micros)
r:32481363 nreturned:10 reslen:2060 17017ms

17037ms Apr  5 16:00:05 m202-ubuntu mongod.27017[808]: Fri Apr  5 16:00:05
[conn16030] query serverside.scrum_master query: { $query: { datetime_used: {
$ne: null } }, $orderby: { _id: 1 } } cursorid:4255253416419073674 ntoreturn:10
ntoskip:0 nscanned:7000376 keyUpdates:0 numYields: 52887 locks(micros)
r:32261752 nreturned:10 reslen:2060 17037ms

17083ms Apr  5 16:29:53 m202-ubuntu mongod.27017[808]: Fri Apr  5 16:29:53
[conn16030] query serverside.scrum_master query: { $query: { datetime_used: {
$ne: null } }, $orderby: { _id: 1 } } cursorid:5345926353689011561 ntoreturn:10
ntoskip:0 nscanned:7000376 keyUpdates:0 numYields: 48591 locks(micros)
r:32203371 nreturned:10 reslen:2060 17083ms
These 5 queries each took more than 15 seconds, and all look pretty similar. Let's find the slowest queries involving sorting; we'll filter on the string "orderby":

egrep '[0-9]{3,}ms$' mongod.log | grep orderby | awk '{print $NF, $0}' | sort -n | tail -n 5
16872ms Apr  5 16:00:26 m202-ubuntu mongod.27017[808]: Fri Apr  5 16:00:26
[conn16030] query serverside.scrum_master query: { $query: { datetime_used: {
$ne: null } }, $orderby: { _id: 1 } } cursorid:6213124304229200070 ntoreturn:10
ntoskip:0 nscanned:7000376 keyUpdates:0 numYields: 51490 locks(micros)
r:32180103 nreturned:10 reslen:2060 16872ms

16965ms Apr  5 15:57:45 m202-ubuntu mongod.27017[808]: Fri Apr  5 15:57:45
[conn16030] query serverside.scrum_master query: { $query: { datetime_used: {
$ne: null } }, $orderby: { _id: 1 } } cursorid:8343223522342664909 ntoreturn:10
ntoskip:0 nscanned:7000376 keyUpdates:0 numYields: 48058 locks(micros)
r:32188994 nreturned:10 reslen:2060 16965ms

17017ms Apr  5 16:30:35 m202-ubuntu mongod.27017[808]: Fri Apr  5 16:30:35
[conn16030] query serverside.scrum_master query: { $query: { datetime_used: {
$ne: null } }, $orderby: { _id: 1 } } cursorid:3380969414085642540 ntoreturn:10
ntoskip:0 nscanned:7000376 keyUpdates:0 numYields: 52433 locks(micros)
r:32481363 nreturned:10 reslen:2060 17017ms

17037ms Apr  5 16:00:05 m202-ubuntu mongod.27017[808]: Fri Apr  5 16:00:05
[conn16030] query serverside.scrum_master query: { $query: { datetime_used: {
$ne: null } }, $orderby: { _id: 1 } } cursorid:4255253416419073674 ntoreturn:10
ntoskip:0 nscanned:7000376 keyUpdates:0 numYields: 52887 locks(micros)
r:32261752 nreturned:10 reslen:2060 17037ms

17083ms Apr  5 16:29:53 m202-ubuntu mongod.27017[808]: Fri Apr  5 16:29:53
[conn16030] query serverside.scrum_master query: { $query: { datetime_used: {
$ne: null } }, $orderby: { _id: 1 } } cursorid:5345926353689011561 ntoreturn:10
ntoskip:0 nscanned:7000376 keyUpdates:0 numYields: 48591 locks(micros)
r:32203371 nreturned:10 reslen:2060 17083ms
Are all the sorting queries this slow, or only this one? Let's look for the fastest sorting queries by using head instead of tail:

egrep '[0-9]{3,}ms$' mongod.log | grep orderby | awk '{print $NF, $0}' | sort -n | head -n 5
15753ms Apr  5 16:29:32 m202-ubuntu mongod.27017[808]: Fri Apr  5 16:29:32
[conn16030] query serverside.scrum_master query: { $query: { datetime_used: {
$ne: null } }, $orderby: { _id: 1 } } cursorid:468013025027691685 ntoreturn:10
ntoskip:0 nscanned:7000376 keyUpdates:0 numYields: 12113 locks(micros)
r:31142862 nreturned:10 reslen:2060 15753ms

15805ms Apr  5 15:59:24 m202-ubuntu mongod.27017[808]: Fri Apr  5 15:59:24
[conn16030] query serverside.scrum_master query: { $query: { datetime_used: {
$ne: null } }, $orderby: { _id: 1 } } cursorid:4493330445843070615 ntoreturn:10
ntoskip:0 nscanned:7000376 keyUpdates:0 numYields: 16742 locks(micros)
r:30448693 nreturned:10 reslen:2060 15805ms

15886ms Apr  5 15:58:45 m202-ubuntu mongod.27017[808]: Fri Apr  5 15:58:45
[conn16030] query serverside.scrum_master query: { $query: { datetime_used: {
$ne: null } }, $orderby: { _id: 1 } } cursorid:6428655199185527636 ntoreturn:10
ntoskip:0 nscanned:7000376 keyUpdates:0 numYields: 15215 locks(micros)
r:30567474 nreturned:10 reslen:2060 15886ms

16039ms Apr  5 16:29:13 m202-ubuntu mongod.27017[808]: Fri Apr  5 16:29:13
[conn16030] query serverside.scrum_master query: { $query: { datetime_used: {
$ne: null } }, $orderby: { _id: 1 } } cursorid:7606044462464616588 ntoreturn:10
ntoskip:0 nscanned:7000376 keyUpdates:0 numYields: 22861 locks(micros)
r:30670479 nreturned:10 reslen:2060 16039ms

16183ms Apr  5 16:27:48 m202-ubuntu mongod.27017[808]: Fri Apr  5 16:27:48
[conn16030] query serverside.scrum_master query: { $query: { datetime_used: {
$ne: null } }, $orderby: { _id: 1 } } cursorid:4359096396969049572 ntoreturn:10
ntoskip:0 nscanned:7000376 keyUpdates:0 numYields: 24743 locks(micros)
r:31557172 nreturned:10 reslen:2060 16183ms
Even the fastest sorting queries are about this slow.


###################################

examining connections in log files

what if problem is connections spikes

grep for accepted connections on Apr 5 and count them per hour. to increase granularity increase dots amount (with 5 dots will be per minutes)
grep connection mongod.log | grep acce | grep -o "Apr 5 .." | sort | uniq -c


grep connection mongod.log | grep acce | awk '{ split($0,a,":"); print $14, $16 ,a[0]}' | sort | uniq -c

find most connections
grep connection mongod.log | grep acce | awk '{ split($14,a,":"); print a[1]}' | sort | uniq -c | sort -n

From the video, the pipeline for grouping by arrival time is
grep connectio mongod.log | grep acce | grep -o "Apr  5 ...." | sort | uniq -c
The first two greps filters the lines we're interested in, the grep -o selects the field we want to group by, and the sort | uniq -c shows the number of times each line occurs. To group by client host instead of arrival time, we'll just select a different field to group by.
grep connectio mongod.log | grep acce | head -n 3
Apr  5 06:39:57 m202-ubuntu mongod.27017[808]: Fri Apr  5 06:39:57
[initandlisten] connection accepted from 127.0.0.1:15991 #8952 (59 connections
now open)

Apr  5 06:40:02 m202-ubuntu mongod.27017[808]: Fri Apr  5 06:40:02
[initandlisten] connection accepted from 192.0.2.2:63301 #8953 (59 connections
now open)

Apr  5 06:40:02 m202-ubuntu mongod.27017[808]: Fri Apr  5 06:40:02
[initandlisten] connection accepted from 192.0.2.2:63303 #8954 (59 connections
now open)
We want the IP address after "accepted from".
grep connectio mongod.log | grep acce | grep -o 'from .*' | cut -f2 -d' '  | cut -f1 -d: | sort | uniq -c
2837 127.0.0.1
 110 192.0.2.10
16120 192.0.2.2
2621 192.0.2.3
2677 192.0.2.4
 126 192.0.2.5
 107 192.0.2.6
 122 192.0.2.7
 133 192.0.2.8
Finally, you can sort this output numerically to quickly see which hosts have the highest connection counts:
grep connectio mongod.log | grep acce | grep -o 'from .*' | cut -f2 -d' '  | cut -f1 -d: | sort | uniq -c | sort -n
 107 192.0.2.6
 110 192.0.2.10
 122 192.0.2.7
 126 192.0.2.5
 133 192.0.2.8
2621 192.0.2.3
2677 192.0.2.4
2837 127.0.0.1
16120 192.0.2.2

