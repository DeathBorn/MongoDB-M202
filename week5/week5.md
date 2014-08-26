Scaling Out with Shards: practical COnsiderations

Vertical scaling is bad when you are growing faster thant technology and it is very expensive

Horizontal scaling - automatic sharding

-Sharding (made of relica sets)
	Apllication does not need to know about sharding)
-Planning
	know your limits?
	make sure you never reach it! - add capacity after ~80% so you have 20% buffer
	how much time does that buy you? 

Which of the following are essential questions to answer in planning a strategy for scaling out with shards? Check all that apply.
	What tolerance do I have for data loss?
	====By what factors should I measure the capacity of my system?
	====How many shards do I need to add to keep my capacity under 80%?
	What is the age of my existing infrastructure and should I consider updating to more reliable hardware?
	====What growth trend does my application follow along my primary capacity metric?


############
Config Servers

how thet operate?

-what are they?
	the brain of the sharded cluster
-Meta data for the Cluster(what is in this cluster, settings, map of shards...)

-Config Servers are NOT a replica set
	they do not sync from each other
-standalone mongod
-they are identical
-They are not all created equal
	the config string assed to mongos (one is more important)
		the first config server in the list is important(because it will stuck on this) (for mongos)

Lose a Config Server
-Lose 1 or 2 config servers
	cluster continues to function (cant update config servers, cant migrate chunks, cant split, but shards will accepts writes)

	to update use 2 phase commit.

1 2 3
C C C
if you loose first then it will have delay because mongos will try to read from first

Changing config servers is hard
-Downtime
-Treat the Config Servers as Critical


Which of the following accurately describe config servers in a sharded cluster? Check all that apply.

	====You can consider config servers to be identical with regard to the data they contain.
	The configure servers form a replica set.
	Requests from mongos processes are distributed across the config servers.
	====Config servers are critical to the health of your sharded cluster.
	Config servers provide automatic failover such that if one goes down, meta-data updates can continue.

##############
The config Database

mongo --nodb

var cluster = new ShardingTest({shards:2, chunksize:1, config:3}); //chunksize 1mb

new tab

mongo --port 30999
sh.status()

use config
show collections
	changelog
	chunks
	databases
	locks

db.settings.find().pretty()
db.mongos.find() //will show mongos connected
db.changelog.stats()  // capped collections
db.changelog.find()

db.chunks.find()

db.databases.find() //shows databases

sh.enableSharding("shardTest")
db.foo.insert({"a":1})
sh.shardCollection("shardTest.foo",{"_id":1}) //oly for testing purposes, because you need to index field

for(var i=0; i<=100000; i++){db.foo.insert({a:i})}

sh.status()  //will show shards and databases

use config
db.chunks.find() //

db.collections.find() //will find collection

Vital collections (checked for consistence between config servers)
-chunkc
-databases
-collections
-shards
-version

hash computed and compared

Find Caveat - NEVER UPDATE DIRECTLY
	use helpers. sh.enableSharding()
	do it through the mongos

#############
Periodic Maintenance on Sharded Clusters

-Rolling Maintenance Methodology (still apllies hereas for the shard)
	replica sets covered
-Added new Components
	the Config Servers (can be backed by mongodump)
		if you want to backup, shut down second or third server because it will be then read only, do backup.
		dont shutdown first because delay
	the Mongos processes (never to compact,no indexes, no backup)

#############
Upgrades on sharded clusters
-Always:Read the Release Notes and the Upgrade Notes
-Order (version 2.4 has strict order)

General Order
1-Secondaries of your Shards First (can be in parallel)
2-Upgrade your Primaries (step Down) (not recommended step down primaries on the same time)
3-Config Servers (can be also first)
4-Upgrade your Mongos processes (must be last, quete critical)

True or false? When upgrading from MongoDB 2.4 to 2.6 there is a strict order in which you must upgrade the mongod, mongos, and config servers. (Note: there is a homework on this subject at the end of this chapter.)
	====True
	False

##############
Services a mongos performs

-primary thing is to route queries to shards
	mongos keep a caches version of configs, looks into it, decides where to sent (to which shard).

Primary Duties
-Routers for requests/ops
-Present Clients with a simgle view of the Sharded DB

HouseKeepingDuties
-Split Chunks(mongos asks, shard says how to split, primaries sends info to configs)
-Coordinates Chunk Migrations (coordinates with primaries)

if you do find().sord()
then two shards sends sorted lists for mongos and then mongos use merge sort to combine these results.


Which of the following are services a mongos performs? Check all that apply.

====Coordinates the process of keeping shards balanced
====Provides an abstraction layer that frees clients from a need to know how the data is sharded
Maintains all cluster meta data
====Merges results from individual shards for queries involving sort()
Maintains indexes for shard keys

##########
Mongos processes do connections pooling

why have a connections pool? because TCP is 3 way handshake. to mitigate this overhead use connection pooling
sending Keep Alive mongos to shards. so network partition and
then socket exception so client gets that
in 2.6 version this thing is rewriten so better 
better to restart

Refreshing the Cache of Mongos (if it is stale)
use admin
db.runCommand({flushRouterConfig:1})

###########
Chunk splitting algorithm

-Chunks are not physical data
	logical grouping/partitioning
	described by the meta data
-SPlit a chunk
	no change to actual data
	changing metadata

Splits(auto)
-heuristic (algorithm)
	mongos->tracks writes to chunks
		20% of the max chunk size (12-13Mb data) when it sees that then
		sends command splitVector -> shard primary (tell me if splitable, if it is then tell me at which points it will be split)
		primary returns a list of split points
		update the meta data to reflect that split
		no data has moved, no data has changed

Which of the following occur when a chunk is split?
	A small split token is placed in the data file
	The data file is turned into two files, but they are at the same location as they were before
	====A split alters meta data only, there is no change to the data itself
	The data file is split in two and possibly moved
	A new chunk is created and new writes go to the new chunk

##########
Manual Split

splitFind()
splitAt()

pass query to splitFind() 
splitAt() pass a specific split point

turn autosplit off -> --noAutoSplit

AllChunkInfo("shardTes.foo")  //javascript

sh.splitFind("shardTest.foo",{_id:ObjectId('24sefg4')})
{ok:1}
//will see split

###########
Jumbo chunks

is one that exceeds maximum size of chunks
usually this is for bad shard key

26 possible values [a-z]
s -> millions - huge ( exceed 64MB, 250,000)
z -> few hundred

marked as "JUMBO" cannont be moved!!!!!!

Which of the following is the most likely cause of MongoDB marking a chunk as jumbo?
	The balancer is not running
	Too many large documents in your collection
	Too few shards
	====A shard key with a cardinality that is too low
	Too many documents in the collection

#########
Manual pre-splirring 

Caveat: DO this before you insert data
When ?
	know domain
	multiple shards up and running
	you want bulk insert
	want to avoid bottlenecks

##########
Manual pre-splitting example: Describing the data

- 10k awb doc size
- 6 million of them
- GUID - 32 character Hex String
- so approx 60GB of data then

-max chunk size - 64MB (default) -> you can reduce 32MB
function GUID() {
	//four characters at the time
	var S4 = function(){
		return Math.floor(Math.random() * 0x10000 /* 65536 */).toString(16);
	};
	return (""+S4()+S4()+S4()+S4()+S4()+S4()+S4()+S4());
}

1920 chunks out of 60GB / 32 MB

- 16 permutations * 16 * 16  -> 16^3 ~ 4096/2 = 2048

########
Actually presplitting the data

for(var x=0; x<16 x++){
	for(var x=0 y<16; y++) {
		//for the innermost loop we will increment by 2 to get 2048 total iterations
		// make this z++ for 4096 - that would give ~30MB chunks based on the original
		for(var z=0;z<16;z+=2){
			//now construct the GUID with zeroes for padding - handily the toString
			var prefix = "" + x.toString(16)+ y.toString(16)+ z.toString(16) + "00000000000000000000000000000"     //29 zeroes
			//finally use the split command to create the appropriate chunk
			db.adminCommand({split:"users.userInfo",middle:{_id:prefix}})
		}
	}
}

use users
sh.enableSharding("users")
sh.stopBalancer()
sh.shardCollection("users.userInfo",{_id:1})

run script

sh.status(true)
//now need to rebalance
sh.startBalancer()
sh.getBalancerState()
sh.status()
	test-rs1 1024
	test-rs0 1025

########
Startng and stopping the balancer

Logistics
-can be disabled (two ways)
	sh.stopBalancer() //recommended // it waits to for balancer to stop 
-start sh.startBalancer()


db.settings.dinf().pretty()

sh.disableBalancing("database.coll") //for specific collection
sh.enableBalancing("database.coll")
db.collections.find()


#############
Running the balancer in a sceduled window

db.settings.update({_id:"balancer"},{$set:{activeWindow:{start:"23:00",stop:"6:00"}}},true)
or use {$unset:{activeWindow:true}}


#######
WHen does a balancer kicks in?
	prior to 2.2 -> difference of 8 or more
	post 2.2 
		total chunks < 20: 2
		20 < total chunjs <80: 4
		total chunks >= 80 :8

##########
How to does the balancer pick chunks
-Moves chunks from the shard with highest chunk count
-Move the chunk with the lowest range
-Move to the least count of chunks having shard 











