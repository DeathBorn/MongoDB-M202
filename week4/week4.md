Rolling Maintenance
-Replica Sets
	at least 3 node without big risk

	taking down a node, performing same unit of work restoring that node
	sufficient

oplog is capped collection
oplog window
	size of the oplog
	rate at whih operations are added
		size operations


shows first last op times
db.printReplicationInfo() 

Methodology

P
S   A
too dangerous schenario, but will still be up

best practive would be
P
S1    S2

then shutdown s2
and replica set is still running

then start S2 without --replSet on different port(because will remove that node from replica set in configuration)

then do some actions
and shutdown and run as initialy


run mongod with mongod user when you do manualy.

--
when we restarted with --replset, wait for S2 to catch up
run rs.status() to check for it

When S2 is running ok
then repeat the process on S1

then run rs.stepDown() in P
this will make it secondary.
then you cna repeat the process on this node.


Which of the following are considered part of best practices when doing rolling maintenance on a replica set? Check all that apply.
	====Perform maintenance on one secondary at a time.
	Ensure your replica set has a minimum of three data-bearing secondaries.
	====Shut down a secondary and restart it without --replSet to perform maintenance.
	====Restart secondaries on a different port when performing maintenance.
	None of the above. MongoDB does not recommend rolling maintenance.

##############
Rolling maintenance use cases
-Upgrading the binaries(yum, apt-get , manual etc)
-index builds (especialy for pre 2.6 versions)(background index build are not possible below 2.6.. on 2.6 and later it happens in background, but sometimes it is better to do even on 2.6 version)
	-buld on secondaries first
	-choose 
		background build on primary
		step down primary repeat process
-compact/Repair (of course you can sompletely resync from scratch)
###########
Can I put a load balancer in front of replica set
-No if you are using replica sets
-yes, with caveats
	-sharding

what are difficulties using load balance
	msut able to do isMaster
	able to interpret read preferences

######
Using load balancers with sharded clusters
there is no shared state between mongos. so getmore on another mongos will be what cursor?.
	two schenarious if it is down or full of connections

affinity
stickiness
source based affinity/stickiness
this must be enabled on balancer to solve upper problem.
if target goes away then destroy clientside connection.
MAJOR CAVIATS because of cursor errors

Summary
-No load balancing possible fro replica set
-limited use for sharded environment
	- must be comfigured correctly
	-must be aware of problems

#################
Driver options: connections

-Generic - most familiar with teh Java driver
-connection timeout
	balancing act
		how busy is the server/cluster
		how quickly do you need to fail
		throw exception or retry? - in java does not retry by default
			back off, seed a delay message back to user, retry in 10s
-Connections per host
	blocking multiplier (can have some blocking, waiting connections)

	be carefull because every connection is 1MB

##################
Driveroptions: socket timeout
-Socket time out (infinite)
	how long to wait for a response on an open socket
	better to fix why it is long before doing upper limit

#################
Driver Optins: high availability ( not in all drivers)

HA- High availability-mongos

MongoClient(localhost:27017)
another way is
MongoClient(localhostL27017,host2:27017,hostn:27017)
but still can raise execption "what cursor" on getmore action.

Suppose your application is making use of a cursor to read a large amount of data from a sharded cluster. The mongos your application is talking to goes down and the application connects to another mongos. Which of the following will occur?
	The new mongos will continue to use the same cursor seamlessly.
	====The cursor will be lost and an exception will be raised.
	The cursor will be reset, and the query will start over at the beginning of the result set when getmore is called next.
	The cursor will skip one getmore call, so some documents in the result set will be skipped.
	The cursor will behave as if it reached the end of the result set.

#####################
Connection Management in replica sets

-informally looked at connections several times
-standalone mongod
-replica set
-sharded cluster

Conncection BUild up progression

C  M
usage - dictates  of connections so you can monitor with mms monitoring)
more spikes can come from other clients

REplica set
P     C    C      C       MMS

S     S

writes goes on P and is checked by MMS
reads can go to S
also can be administering connections
finally administrative connections from replica set within (heartbeat and other)

primaries will have the most connections so it must be on focus
- every connections per 1MB
- max limit
	-ulimit(file descriptors)
	-20000 cap
	-memory

Which of the following are types of connections you might find in a replica set? Check all that apply.
	====Heartbeat connections between members
	====Connections from secondaries to their sync source in order to tail the oplog.
	====Connections from secondaries to their sync source to keep track of where they are in the oplog
	====Administrative connections from drivers to all members
	====Connections from clients reading from secondaries


#####################
Connection management in sharded clusters

Prior to MongoDB 2.6, the upper limit for maxIncomingConnections for a mongod or mongos was 20,000. Starting with MongoDB 2.6, this is no longer the case. There is no longer an upper limit on what value you can choose as your maxIncomingConnections.

client connections
administrative connections 
if 75% is utilised then it is time to worry

how to limit things here
there is a parameter in mongos
--maxConns = 100    
this limits at mongos layer


Suppose you have an application backed by a sharded cluster. Each shard is a three-node replica set in which all nodes are data bearing. Your application is at 80% utilization for the maximum number of connections. Which type of process is the bottleneck in this scenario?
	Mongos
	Config server
	====Primary
	Secondary
	Client

##############
Formula for maximum Mongos connections

	ulimits
	memory
	cap (20,000)

max connections on the primary?
	maxPrimaryConnections(20,000)
Number of mongos processes? 
	maxMongos (100) 
Number of Secondaries in ReplicaSet? 
	maxSecondaries (2) 
Other Connetions?
	numOther (2) (MMS or other)

FORMULA
(maxPrimaryConnections - (numSecondaries x 3) - (numOther x 3)) / numMongos

so maybe 199.88 round this down to 190 which maximum connections  and about ~90% capacity


In mongodb 2.6, what is the hard limit on the maximum number of connections that a server can have?
	10,000
	20,000
	30,000
	40,000
	====There is no limit

##############
Read preferences (for availability)

P
S1   S2

default read preference is primary

if you read from secondary then there is a problem of eventual consistency (delay of catchuping) 

other options:
-primary prefered
-secondary (only)
-secondary prefered
-nearest

analitics can use secondaries

###############
Rollbacks

simulating rolback on adam blog

p
s      a
writes goes to p then will be x of writes on p but not replicated
and s becomes P. when you fix this problem and bring online p which became S. it tries to rejoin and catchup but it noticed that there are some writes after common point. it goes into rollback state - removes these operations, dumpt out data to disk and rejoin.
but it happens it if data is <300MB

but if >300MB then you need to resync data, do it manually

dumped data will be mongorestore compatible bson format.


mongo --nodb
var rst=new ReplSetTest({name:'testSet', nodes:{node1:smallfiles:"",oplogSize:40, noprealloc:null, node2:{smallfiles:"", oplogSize:40, noprealloc:null}, arb:{smallfiles:",oplogSize:40,noprealloc:null}}});

if you try to start name with an "a" then it will assume it will be arbiter

rst.startSet()
rst.initiate()

=
mongo --port 31000
rs.status()
for(var i=0;i<10000000;i++){db.test.insert({d:1})}

ps
kill -SIGSTOP 2673
kill -SIGINT 2672; kill -SIGCONT 2673

mongod --oplogSize 40 --port 31000 --smallfiles --rest --replSet testSet --dbpath /data/db/testSet-0 --setParameter enableTestCommands=1

resplSet rollback 4 n:612 <- how many operations

find data dumped

test.rollback.timestamp.bson
you can examine data or use mongorestore

Which of the following are true of automatic rollback? Check all that apply.
	====Automatic rollback occurs only if rollback is less than 300MB.
	====Manual intervention is always required to restore the data.
	Rollback can't occur if w="majority".

##########
Other MongoDB states
11 states that a mongod instance have
0 - STARTUP 
	parsing the config document (cannont vote)
5 - STARTUP2
	threads forked for replication elections,what to sync from (cannont vote)
1 - PRIMARY 
	only member accepting write ops (can vote)
2 - SECONDARY
	replicating data (can vote)
3 - RECOVERRING (can vote)
	ROLLBACK
	Catching up to the set
4 - FATAL 
	bad states - Encountered serious error - unrecoverable (cannot vote)
6 - UNKNOWN
	never connected successfully (maybe network or something,firewall, cannot vote)
7 - ARBITER
	just to vote
8 - DOWN
	completely inaccessible - cannot vote
9 - ROLLBACK
	can vote
10 - SHUNNED
	was once on the set, removed - cant vote




















