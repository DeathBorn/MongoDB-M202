Homework: 5.1: Pre-splitting data
----

For this assignment, you will be pre-splitting chunks into ranges. We'll be working with the "m202.presplit" database/collection.

First, create 3 shards. You can use standalone servers or replica sets on each shard, whatever is most convenient.

Pre-split your collection into chunks with the following ranges, and put the chunks on the appropriate shard, and name the shards "shard0", "shard1", and "shard2". Let's make the shard key the "a" field. Don't worry if you have other ranges as well, we will only be checking for the following:

  Range  / Shard
 0 to 7  / shard0
 7 to 10 / shard0
10 to 14 / shard0
14 to 15 / shard1
15 to 20 / shard1
20 to 21 / shard1
21 to 22 / shard2
22 to 23 / shard2
23 to 24 / shard2

You may have other chunks with other ranges (you undoubtedly will if you are solving the problem correctly!) but we will only be checking for these. Keep in mind that if your balancer is on, it may move chunks around if it detects an imbalance. Also, remember that you can check the state of your sharded cluster by calling sh.status(true).

Start three stand alone servers, three config servers and a mongos:

# Kill any running process
pkill mongod
pkill mongos

# Remove our folders
rm -r shard* config*

# Create our folders with the following formats
#    shard0-0
#    config-0
mkdir {shard{0,1,2}-,config-}{0,1,2}

# Start our stand alone servers
mongod --shardsvr --port 30101 --dbpath shard0-0 --logpath shard0-0/shard0-0.log --smallfiles --oplogSize 40 --fork
mongod --shardsvr --port 30201 --dbpath shard1-0 --logpath shard1-0/shard1-0.log --smallfiles --oplogSize 40 --fork
mongod --shardsvr --port 30301 --dbpath shard2-0 --logpath shard2-0/shard2-0.log --smallfiles --oplogSize 40 --fork

# Start our config servers
mongod --configsvr --port 30501 --dbpath config-0 --logpath config-0/config-0.log --smallfiles --oplogSize 40 --fork
mongod --configsvr --port 30502 --dbpath config-1 --logpath config-1/config-1.log --smallfiles --oplogSize 40 --fork
mongod --configsvr --port 30503 --dbpath config-2 --logpath config-2/config-2.log --smallfiles --oplogSize 40 --fork

# Start our mongos server
mongos --port 39000 --configdb precise64:30501,precise64:30502,precise64:30503
After these all start up, run the following commands when connected to the mongos server:

use admin
db.runCommand({"addShard": "precise64:30101", "name": "shard0"})
db.runCommand({"addShard": "precise64:30201", "name": "shard1"})
db.runCommand({"addShard": "precise64:30301", "name": "shard2"})
sh.enableSharding("m202")
sh.shardCollection("m202.presplit", {"a": 1})
sh.splitAt("m202.presplit", {"a": 0})
sh.splitAt("m202.presplit", {"a": 7})
sh.splitAt("m202.presplit", {"a": 10})
sh.splitAt("m202.presplit", {"a": 14})
sh.splitAt("m202.presplit", {"a": 15})
sh.splitAt("m202.presplit", {"a": 20})
sh.splitAt("m202.presplit", {"a": 21})
sh.splitAt("m202.presplit", {"a": 22})
sh.splitAt("m202.presplit", {"a": 23})
sh.splitAt("m202.presplit", {"a": 24})
sh.stopBalancer()
sh.moveChunk("m202.presplit", {"a": 0}, "shard0")
sh.moveChunk("m202.presplit", {"a": 7}, "shard0")
sh.moveChunk("m202.presplit", {"a": 10}, "shard0")
sh.moveChunk("m202.presplit", {"a": 14}, "shard1")
sh.moveChunk("m202.presplit", {"a": 15}, "shard1")
sh.moveChunk("m202.presplit", {"a": 20}, "shard1")
sh.moveChunk("m202.presplit", {"a": 21}, "shard2")
sh.moveChunk("m202.presplit", {"a": 22}, "shard2")
sh.moveChunk("m202.presplit", {"a": 23}, "shard2")
sh.startBalancer()


















Homework: 5.2: Advantages of pre-splitting
----

Which of the following are advantages of pre-splitting the data that is being loaded into a sharded cluster, rather than throwing all of the data in and waiting for it to migrate?

1.  Data can get lost during chunk migration. Pre-splitting allows you to avoid that.
2.  You have more control over the shard key if you pre-split.
3.  **You can decide which shard has which data range initially if you pre-split the data **
4.  **Migration takes time, especially when the system is under load. Pre-splitting allows you to avoid that.** 


Homework: 5.3: Upgrading a sharded cluster to 2.6

Which of the following are required when upgrading a sharded cluster to MongoDB 2.6? Check all that apply.

1.  **If your MongoDB deployment is not already running MongoDB 2.4, upgrade to 2.4 first.** 
2.  **Upgrade all mongos instances before upgrading mongod instances.** 
3.  Upgrade all mongod instances before upgrading mongos instances.
4.  **Disable the balancer.** 
5.  Upgrade the secondaries on all shards before upgrading any primary.


Homework: 5.4: Automatic chunk splitting
----

In a sharded cluster, which of the following can keep large chunks from being split as part of MongoDB's balancing process? Check all that apply.

1.  **Frequent restarts of all mongos processes relative to the frequency of writes**
2.  **If there are not enough distinct shard key values**
3.  **If one of the config servers is down when a mongos tries to do a split**
4.  If the average size of the documents in the chunk is too small
5.  If the number of shards in the cluster is too small

Explanations:

1.  
2.  
3.  If at least one is down then config servers will become readonly, and primaries could not be able to change metada - can't split [source](http://docs.mongodb.org/manual/core/sharded-cluster-config-servers/#config-server-availability)
4.  
5.  
