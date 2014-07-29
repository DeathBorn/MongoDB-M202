CHAPTER 2: SYSTEM SIZING AND TUNING


#MongoDB Memory Model

- uses Memomyr Mapped Fieles (mmap()) (create virtual memory and gets missing data lazyly into memory)
- Mapped into virtual memory
- accessed lazily as needed -> phycial memory

Until physical memory gets full, then kernel must to find page to discard, allocate for new data

(Discarding Pages)
Kernel decides by algorithm LRU - Least Recently Used  (longest siting untouched page in memory is discarded)
# REsident memory in MongoDB
- Working set
		Portion of data that is accessed most often
			-indexes
			-Subset of data
		How to figure out working set size?
			good knoweledge of your data (and indexes)
			average document size
			average size of index bucket, key
- REsident Memory
	- FS cache
	- process
- Restarts and Dropping Caches
# 
Working must be set enabled explicitly by command
	db.serverStatus({workingSet:1});

db.serverStatus
however this is only an estimate
	workingSet
		note
		pagesInMemory: 77121 (~340megabytes)
		computationTimeMicros
		overSeconds

	mem
		resident memory: 1052

Pages in use mongodb
	frequently accessed pages are workingSet in physical memory
	Resident - all pages accessed and which are in physical memory

# 
What offers the best performance in terms of Resident Memory?
	Working data set fits into resident memory
	Indices fit into resident memory
	Either indices or the working data set fits into resident memory, but not both
	====Full data set fits into resident memory
	Indices are larger than working memory

# Resident Memory as a Metric
- For higher than "working set" potentially (very little accessed)
- For lower than "working set" 
	- journaling 

		data File -(sync data)> mmap() shared view ->(remap) private view
		journal file 
		private view
		remap (comits, writes)


		MongoDB
			100%reads all data is in resident

		app changes then


		MongoDB
			95% read
			5% writes (mongod does remaps (part of journaling) because of writes,so you see lower resident memory, so you need to know what is happening)

	too low <50% resident memory is not good


#  Process restarts

- mongod, mongos - restart
	resident memory bicomes biggers and drops and climbs again. when restarting data still maybe in memory so resident memory is quicker to become bigger
	no hard -age fauts => dta still in memory
		hard -data was read from disk (very slow)
		soft - data already in memory, reassigned (quicker, changing a point only)

- reboot -> all caches envalidated (OS restarted so lots hard page faults, lots ios)

How do I invalidate without rebooting?
	proc is respondible for caches
	sudo ls /proc/sys/vm/drop_caches
	echo something to that file to do with caches
	0
	1 - it is enouth
	2 
	3 drops everything
	echo 1 > /proc/sys/vm/caches
	another methods
	sysctl -w vm.drop_caches=1

# Pre-heating Data
how to ready a node for adding (avoids locks and )
	- touch command (blunt option)
	- targeted queries (a bit better)

#  
create  collections with simple for loop about 200000 documents
so run stats
db.heatingTEst.stats(1024*1024)

touch averythin in memory
db.runCommand({touch:"heatingTEst", data:true, index:true}) (both for index and data)
look into mongostats

touch command - reads into the FSCache
how to prove that?  db.heatingTEst.find({}).explain() will give if it is in ram then fast
then memory stas you will see change in statistics


# Preheating with targeted quiries
db.heatingTEst.ensureIndex({date:1})
db.heatingTEst.find({date:($gt:ISO("2014-12-17"))}).hint({date:1}).explain()

Why would you want to pre-heat a specific subset of a MongoDB dataset? 
	Your dataset is twice the size of your resident memory, and you access the data more or less randomly, but you want to pre-heat the other half that is not in memory.
	You have been using the dataset extensively, but it has not yet been pre-heated.
	====You have not used a particular dataset, but you expect to be using that dataset in the immediate future.
	Your index, which is used in every query, fits in memory, but your dataset, if loaded, would push your index out of memory.
	You are working with dataset A, and plan to continue working on it for the next 4 hours. After that, you plan to work on dataset B. You want to pre-heat the data in set B now so that it will be ready when you get to it.

# Storage and Disk Considerations
- Spinning Disks (traditional)
- Network Based Storage (NFS,Amazon storage)
- Solid State Disk (SSD)

# Spinning Disks
very slow on random reads/writes
but cheap
	- Formulae
	- What use cases in MongoDB suit?
		- Capped Collections(oplog)
		- Journal
	- To speed have as much in memory as possible
# Network storage
- NFS 
- Abstracted Sevices (present as block devices) (backed by different physical devices)
	- EBS
	- VMDK(vmware)
- Adds latency (because network)
# SSDs
- relatively immature
- Significantly faster (no seeking)
- Variable performance (firmware, controller) (be carefull)
- Consumer(MLC) Vs Enterprise(SCC) type of SSDs (wearing is much less on Enterprise )

You don't have enough SSD's to put your full data set on them; some must go into spinning media. Which of the following should be placed on spinning media in a read heavy system?
	====Capped collections
	====The oplog
	A random selection of collections
	The smallest collections in your database
	Documents that are likely to be moved when updated

# RAID
RAID 0 is bad no redundancy but speed is better
raid 1 redundancy is but loosing disk capacity
raid 10 stripe accros and mirroring. fast and redundant (recommend for ebs) however 50% storage penalty
Amazon Web Services - MongoDB NoSQL  

RAID Complications:
- Backups
- Snapshots

# System level Tuning
NUMA - non unified memory architecture
cpu gets penalty for accesing memory from other cpus memory
hard to control so it is not supported by MongoDB
so you cna disablei n bicomes
or Set interleave policy, disable zone reclaim


Which of the following is a potentially bad system configuration for MongoDB?
	NUMA System using MongoDB init scripts
	NUMA System with NUMA disabled in BIOS
	====NUMA system running mongod manually
	non-NUMA system running mongod manually

# Filesystems and Options
- ext4, XFS (not recommended ext3 because does not gave fallocate() support)
- ext3, or an old kernel - fill with zeroes (gona take seconds, so slows down)
- ZFS, btrfs (not supported or recommended, because little test)

Options for ext4, XFS
- mount option: noatime 
	/etc/fstab
		ctime
		atime -any access (includes reads)
		mtime - 

Options for XFS (better for bigger files)
	-  aligment
	-stripe width (chunk size) 

Which of the following are recommended filesystems for use with MongoDB?
	ext3
	====ext4
	====XFS
	ZFS
	Btrfs

# NFS
- NFS does not play well with journaling
- put journal somewhere else (local disk or something)
	- so it makes complicated Backups
- Separate Recommendations (noatime, nolock in fstab)
better to avoid

Which of the following is the recommendation for using MongoDB on NFS?
	When using NFS ensure that your journal files and data files are stored on the same volume. (if you must then this answer)
	====Avoid it if you can.
	NFS is strongly recommended for use with MongoDB.
	NFS significantly helps write performance.

# Swap
Allocate Some Swap
	OOM Killer Avoidance
	Process with highest Mem Usage usually is Mongodb so it is bad

Why should you allocate swap space when using MongoDB? Check all that apply.
	For use by MongoDB (generally mongodb does not use swap space)
	====To keep the kernel from killing MongoDB when running short on memory
	====To give you as an administrator greater control over how memory is used on a system running MongoDB

# Readahead

when reading 4K page then read additional more sequentualy to put on cache
- The number of extra sectors to be read on for any disk access 

to view that info
sudo blockdev --report
interesting size RA - exmaple 256 = 256* 512 byte sector  so extra 128KBytes into FSCache
If you have Good Locality then this is good, because we will have in memory. for Oplog, Capped Collections you can even increase
- Seeks = Expensive

-Random Access Data Pattern
-SSD
then better very small readahead because FSCAche will be bigger and leave much less space for mongodb
-Priority for Memory Efficiency
- lower Readahead
-How low? 8K - 16 sectors
32 sectors are recommended


Which of the following can readahead settings effect? Check all that apply.
	====The efficiency of your data storage in memory 
	Seek times
	Data locality
	====How often you access disk 
	Data access patterns


Increasing readahead will give the least increase in performance in which of the following scenarios?
	Data to be read from spinning disks
	Data to be read from optical storage
	====Data to be read from SSDs (no benefit at all)
	Sequential documents that are often accessed together
	Documents that are accessed randomly


# Production Notes
- dont use huge pages
- read them, read them often

# CPU considerations
- user  CPU
- system CPU

User CPU (mongodb user)
	- count, distinct, sorts (big cpus)
	- iterations are key
	- clock speed - quire important 
		clock speed decrease is bad. 3.4Ghz -> 2.4 Ghz. slows down.
	- Server Side Javascript (including Map Reduce)
	- Intensive Aggregation Framework
	mongo version 2.4 + -> V* - has paralelism better (not single threaded)
				- more modern upt odate
				- still single Javascript lock

System CPU 
	- Scanning large areas of Memory
	- malloc() -> tcmalloc() change was better.
	- large in-memory sorts

	-Generally CPUs not a primary scaling factor
	-MongoDB will use multiple cores when possible
	-Some functiosn and usage patters are still CPU intensive, faster = btter in those cases

Which of the following scenarios might cause increases on the CPU usage metrics reported in the MMS hardware tab? Check all that apply.
	====Saturated I/O
	====Large counts or distincts
	====Complex MapReduce operations
	====Large in-memory sorts
	====Aggregation queries

# how mongodb uses disk
Disk capacity
by default will create 64MB then 128MB, 256MB ---- 2048MB max dobbling and last file always empty - preallocated
as soon as page comes into that preallocated file, new preallocate is done

Preallocation
- Data file Preallocation 
- Journal is also preallocated
	1GB 1GB 1GB
- Oplog -> b default 5% of free space


Example
100GB disk
	oplog -> 5GB
	journal - 3GB 
	1 db 1da -> 64MB + 128MB -> 192MB
	So 8192MB in all

	with nojournal and otheroptions then
	oplog -> 1GB
	journal - 384MB 
	1 db 1da -> 16MB + 32MB -> 64MB
	So 1456MB in all

- oplogSize = 1024  --oplogSize=1024
- smallfiles - true  --smallfiles

Arbiter
	no data here (but it does not know that it will be liek this)
	- preallocation - journal - 3GB
	- local DB 192MB
	if you know that it will be an arbiter always then run with --nojournal

Deletes, Moves
does not shrink files 
moving document if it does not fit then it can cause preallocating new file

Which of the following are settings you should consider when it\'s appropriate to reduce the disk space a mongod process consumes.
	--nodisk
	====--smallfiles
	====--nojournal (only for non production or arbiters)
	====--oplogSize (for replica sets)
	--noDataFiles

For which replica set members is it appropriate to use the --nojournal option?
	primary
	secondary
	hidden secondary
	priority 0 secondary
	====arbiter

# Reclaiming Disk Space
How to reclaim when not efficient use
- Compact Command  (like defragmentation)
	within the data files
	does not shrink/delete existing data files
- for Standalone Instance
	Repair Command  (Rewrite data from scratch, tries to recover from corruption as the last resort) (similar to fsck, so it is not very safe)
		data integrity is ok
		Repair Caveats
			Data Soursce myst be good
			Up to 2x the Disk Space (because rewrite)

- Replica sets
	take secondary delete all data in dbpath. so it will become like new node and then get all data from primary. So you do not need 2x Disk Space. 
		Downside is lots of network operations

sumarize
	Repair
	Resync (Replica Sets)
	Compact -> does not reclaim space on disk

Which of the following will enable you to reclaim disk space? Check all that apply.
	Using the compact command
	Using the resync command with a standalone mongodb instance
	====Using resync with a replica set
	====Using the repair command
	Using the reclaim command

# Monitoring, Strategies
-MongoDB does not track free space on disk
-MMS Monitoring does not track free space
-No warning/erorrs until no space left

So use some disk monitoring tool.

Notes
	- Smallfiles 
	--directorydb; directorydb=true
	dbpath/journal
		dbname.ns 
		dbname.0 16MB
		dbname.1 2x
		....

True or false. MongoDB tracks disk space usage.
	True
	====False

# Segregation of Resources
12 cores 512GB RAM, SAN , SSD
so if my working set is 80GB - so its is like 5 proceses
benchmark I/O -> 4x processes  (each process has 15000 connections = 60000 concurent connections so then it is  60GB space of memory you need) so it is hard.
if lots of connections -> file descriptors

take beefy server and benchmark workin set, resident set footprints - GOLDER RULE

Note: Very touth to isolate processes, set limits


If you are going to run multiple mongod processes on the same host, which of the following should you do?
	====Benchmark memory usage under production load
	====Measure the number of file descriptors used by a process under production load
	Follow MongoDB documentation guidelines for isolating processes and setting limits
	====Benchmark IO usage under production load

# Virtualization
can allow easy migrate to new host
gives flexibility.

Note: 
	-VMWare - Memory Balloning- is not very good so shoudl be disabled

#Containers(zones/jails)
Mongodb in container not wll tested
despite being araund 
	relatively immature in Enterprise
starting to changes
	Rapidly evolving (Dakev)

why would you need containers? instead install multiple os, share same os, libraries and you can set cmae groups cgroups. but you cant move containers
more unknowns

# Replica Set Sizing
How many members?
Any hidden/specila members
Configuration options?

limits
7 voting members (max limits)
12 total members (max limits) because too messy pings and connections 

ussualy 3 replica sets (P,S,S)(P,S,A)


A three node replica set with a primary and two data-bearing secondaries is the most common deployment. What are some reasons for this?
	====Provides a sufficient guarantee of high availability and distaster recovery for most applications
	MongoDB runs most efficiently in this configuration
	====Enables you to pull out a single secondary temporarily (e.g., for an upgrade) without downtime or too much risk
	The cost of production servers capable of running three mongod processes currently is something of a sweet spot in terms of price
	This is the only configuration that permits three voting members and guarantees some redundancy

# Going more thab 3 nodes
why to go more?
-hidden nodes (never become primary, not visible to the driver)
	analytics
	backups
-lesser nodes (low capability nodes)
	similar to hidden nodes
	not as capable
	user as backup Soursc
-locations 
	multiple data centers

Which of the following are common reasons some deployments have more than three replica set nodes?
	====So that they may include a node purely for analytics
	====To include a node for backups
	Because resources will permit additional nodes
	====To distribute a replica set across multiple data centers
	====To include a less powerful node you do not want clients to access, but require for some other purpose


# Distributed Replica sets
new nodes can sync from other secondary

Replica Set chaining- secondaries choose from where to sync to reduce overhead from primaries.

eventual consistency and asincronous mongodb
how to change from where to sync?
	use commands

From where do replica set secondaries sync by default?
	From the primary
	====From the nearest member
	From a secondary in the same data center
	From a randomly selected member
	From the nearest member unless that member is already syncing from a secondary

# replica set syncing demo
rs.status()

syncingTo field is interesting
(cheat and create test replica set)

mongo --nodb      
var rst= new ReplSetTest({name:'testSet',nodes:3})
rst.startSet();

so it will initiate test replica set. lots of longest
so then

mongo --port 31000
rs.status()  1 primary 2 secondaries. both syncing from primary, lets change for chaining purpose

mongo --port 31002
rs.syncFrom("education.local:31001") tell to sync from secondary (can see error but thats ok)

rs.status()


by default chainingAllowed is true
to change that
var cfg = rs.confg()
cfg.settings = {};
cfg.settings.chainingAllowed = false
rs.reconfig(cfg)

but other will change only when they need to.

drawback of syncing from primary is lots of loads. so balance with chaining





do not set votes for replica set members

