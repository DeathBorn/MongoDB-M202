Disaster REcovery
- More complex than you think
- Trade off (service downtime,degradation, how much you are willing to tolerate, uptime, availability) (how much a second costs?)(ideal and realit)
	
#----------------------
availability over the year
Reference Numbers
30 seconds = ~99.9999 %
1 min =99.9998 %
1 hour = 99.9886 %
1 day = 99.7260 %

> 99% availability

Depends on a lot of things. every hardware, providers are backed, ups, cooling and so on

In a year, what is the maximum number of days of downtime you can have and still claim >99% uptime? 
	Less than 1
	1
	2
	====3
	4

############
Downtime tolerance matrix

Tolerance for: DataLoss Downtime Reduced Capacity
group 3
	1) low low low 
	2) low low high
group 2
	3) low high low
	4) low high high
group 1
	5) high low low
	6) high low high
	7) high high low
	8) high high high

caching system (high data loss tolerance)
batch processing system (maybe high downtime tolerance between some hours)
willing to accept if get high latency if some things go down (high reduced capacity tolerance)

What are the key criteria for deciding your disaster recovery requirements?
	====Tolerance for data loss
	====Tolerance for downtime
	====Tolerance for reduced capacity
	Tolerance for CPU-intensive queries
	Tolerance to iocane powder

#############
Systems with low tolerance for downtime and data loss (group 3)
	- ideal -> no downtime, No data Loss
	-Variable -> Capacity (are you ok with latency)
	-Guiding Principles 
		-> Avoid Data Loss (corruption, error)
		-> Avoid Downtime

Which of the following systems might have a low tolerance for both data loss and downtime? 
	Cache in front of the system of record 
	Data storage for a system that need only deliver information once a month
	====Online ad servers
	A system with several redundant servers and low concern for consistency
	A system using MongoDB only as an in-memory cache in front of a relational database



######
Can I address downtime and data loss with just 2 nodes?

first lesson

Replica set with 2 nodes
Primary and secondary

1 Down time is actualy worse scenario
	Elections problem, because you need strict Majority > 50%. if you have 2 nodes then there is no majority.
	doubled downtime chance
2 Data Loss
	it is improved, but it is violated. Hovewer you can change votes  but it is not recomended. 

so add Arbiter at least.
1. Known configuration
2. Avoiding Data Loss
3. Avoiding DownTime

What conclusions regarding downtime and data loss should we draw in considering 2-node replica sets vs. single node deployments in MongoDB? Check all that apply. 
	====Chances for downtime are increased (from writes perspective)
	Chances for downtime are decreased
	Chances for downtime are unaffected
	Chances for data loss are increased
	====Chances for data loss are decreased
	Chances for data loss are unaffected

##############
Avoid Data Loss  - yes
Avoid Down Time  - no

Datacenters
DC1 and DC2
P        S
S/Arb

If we loose DC1? so bad, no automatic election because of 1/3 majority. you need to recover manualy.
you can have external scripts
it could be 6 hours. if somebody will be there then could be 5 minutes.
So two data centers arent enouth
------
DC1     DC2
P        S
S        S

DC3 (can be in cloud)
Arb

then
Avoid Data Loss  - yes
Avoid Down Time  - yes

known bounds
majority is gained


###########
Considering capacity with downtime and data loss addressed


DC1    DC2
P        S
S        S

DC3
Arb


variable here: capacity

if D1 is prime site "hot"
and DC2 is backup "cold"
then you need good capacity then you need to make sure that DC2 must have same capabilities like DC1
if not then DC2 can be on Virtual machines or less capacity.



DC1    DC2
P        S

DC3
S

variable here: capacity


deploy and make it fail

test even diesel generators frequently enougth

####################
second group
High tolerance for downtime
Low tolerance for data loss


deployments:
-Office-based applications (time frame tolerance)
-batch processing


C1    DC2
P        S

DC3
Arb

better for data loss mitigation, but much better to have DC3

C1    DC2
P        S
S        S
Arb


Which of the following systems might have a low tolerance for data loss, but a high tolerance for downtime? 
	The navigation system used by automobiles manufactured by Ford
	====A courseware system used during the teaching day at an elementary school
	The blogging system used by the London Times
	A retail banking transaction system
	Gmail


############
final group
Hig tolerance for Data Loss (rare scenarious)
(data is not so important)
use case

SoR(System of Record)
using MongoDB as caching

then even low requirement for downtime

should caching use journaling? Yes - garantees consistency, 

Disaster recovery strategy will be dependent

- Downtime
	Replica Sets three datacenters if you need no downtime
	if high tolerance then enouth standalone nodes or 2DC replicasets
-Reduced 
	tolerance - deploy on VM in Disaster recovery sites
	no tolerance - identical deployment in each DC


#####################
Sharded Clusters

- Made up of replica sets
- mongos process - stateless (you can deploy as many as you want, better not to centralize mongos)
- config servers - vital servers. only 3 servers. over 3 DC's
	what to do if you have more than 3 DC's? still limitation. just pick


If you have 5 data centers across which your sharded cluster is distributed, how many data centers will be without a config server?
	0
	1
	====2
	3
	4

#####################
Backup Strategies
how to get out of failures - backups

-how quickly you recover
-how completely you recover
-Feed into DR strategies

components here: cost, busines requirements, performance, disaster recovery

Are Backups a good Idea? YES, but they are not free


-how quickly can i restore
	does that meet my requirements?
-Test often your backups if good connection, time, performance, bugs, compression - restore and run from backus regurarly
- Coordinated Backups are difficult
	distributed systems
	once restored if you get consistent view !!!


With regard to backing up your data, which of the following questions should you take time to consider?
	Do I need to do backups?
	====How quickly can I restore?
	====What complications are distributed systems introducing?
	====Is my test strategy catching errors in backups?


###################
Filesystem-based backup strategies
1 - Shutdown or fsyncLock DB
	-copy files off - scp, cp, rsync - quite safe, but brings downtime

	P
	S    S -> fsyncLock ir shutdown second secondary and do backup, then turn on and catchup

2 - Filesystem Based Snapshot
	-point in time guarantee
	-must include journal (for a database to be in consistent state, because it allows to recover)
	-not free - brings i/o overhead (check if you are able to do that)
	-tools - LVM, EBS (amazon), Netupp.....

What impacts might a snapshot based backup strategy have on a live system? Check all that apply.
	====Slower operations
	You will have to shut your system down to do it
	====Slower disk I/O
	====It's not free
	There are no obvious impacts

#####################
Filesystem-based backup when using RAID (a striped disk set)

-introduces problems , complexity
	not impossible
-LVM ontop of the RAID
-Freeze the data
	-Fsync Lock, shutdown
	-xfs-freeze(makes read-only disk, so makes errors for errors)


######################
Restoration Caveats
-Speed is key
-Test Regularry(full process)
-EBS Snapshot Restores
	magic happens here(lazily loaded blocks-slow)
	so do some pre-heating - touch or targeted find and explain or dd program

######################
Backup: MongoDB Tools

mongoexport/mongoimport - Not suitable
	because uses JSON, you loose datatype precision

mongodump and mongorestore are better 
	use only for small databases (because time,slow if it does not fit in ram, does not dump indexes)
	use for config databse
	use for admin database

mongodump
-sharded environment -> defaults to secondary reads 
- walks through _id index and fetch data by that _id (_id index is used) it guarantees no dublicates
-dont need a running database
	mongodump can acces data files directly
-indexes are not dumped(disadvantage) (but index descriptions are dumped)
- --oplog (dump part of oplog out)
- --forceTableScan -> use with caution! (index _id will be skipped and collection will be scanned, so may dump the same documents multiple times)
	(maybe required to do if you id is custom, index keys are too big)

mongorestore
-it can use on data fiels directly
- --oplog -> partial oplog dump
	--oplogReplay (replays oplog operations, oplog operations are idenpodent so it wont affect multiple replay of same operations
	--oplogLimit (allows to replay with timestamp)


When is it appropriate to use mongodump for backup?
	====For small data sets
	====For config servers
	In large production systems

####################
MMS: BAckup

MMS Backup Agent
it will bring continous backup into cloud service
Agent uses replication oplog and relates over ssh

MMS Backup Service snapshot every 6 hours for 2 days
dailies for 1 week
weeklies for 1 month
monthly for 1 year

Point in time recovering

you can backup replica sets or sharded clusters
cannot backup standalone instances

MMS Backup soft is on MongoDB Enterprice Subscription

Security and encription
transfered over SSH encrypted
but not encrypted at rest
when restores - 2 factor authentication

True or False: You can prevent certain collections from being backed up in MMS?
	====True
	False

###################
MMS: Backup Installation

go backup tab





