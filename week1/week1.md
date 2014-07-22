################################################################################
# chapter_1_performance_and_monitoring

#-------------------------------------------------------------------------------
# handout

# install vagrant
# install virtualbox
# vagrant init
curl -O https://university.mongodb.com/static/10gen_2014_M202_July/handouts/chapter_1_performance_and_monitoring.zip
unzip chapter_1_performance_and_monitoring.zip
cd chapter_1_performance_and_monitoring/spinning_up_a_vm_with_vagrant_unix/MongoDBU
vagrant up

#----------------
++++How many MMS angents need?  >1, or 2 (redundance)
Host down alert - from agent perspective (bad network,firewall or something if hosts are up)
Agent diagnostics - where to look?  
	local logs
	MMS - Agent logs tab
	log in into agent, use mongo shell to relevant host or telnet

Reasons - network ussualy
#-----------------
ulimits - system limits. bad when too low. need to increase. there script to check all hosts.
++++ulimit problem -the resources available to mongod may be too limited
#-----------------
# 1) method to increase ulimit for specific proceses
	ps aux | grep mongo
	#got process id (max open files)
	cat /proc/proc_id/limits
	nano /etc/init.d/mongod
	# uncomment ulimit
	restart mongo
# 2) permanent for mongod user
	sudo su -s /bin/sh -c 'ulimit -a' mongod   #show curent limits
	/ect/security/limits.conf
		mongod 	- nofiles 64000
#--------------------
#Enable hardware stats
sudo apt-get install munin-node
sudo service munin-node start 
telent localhost 4949
list
fetch cpu
fetch iostat

# look into stats
#---------------------------
# Correlated stats
opccounters - lock - background flush avg
non-mapped virtual not usable by data, sorting in memory, map reduce. with cursors go together
----
page faults (not, small writes), non-mapped virtual memory (not, small amount connections), memory (if small enough then it can fit in memory)
----
Below are several groupings for stats you can view in MMS monitoring. Select those groups containing stats that are usually correlated with one another.
	opcounters, lock% (lots operations so lock)
	lock%, journal stats, flush average (how much journal commited, how much taking to sync to disk)
	non-mapped virtual memory, connections (each connection gets 1MB)
	cursors, connections (connections has cursors)
# -------
Dashboards
	create
#--------
Replica Sets
What do you see when you click on a replica set name and enter the replica set view?
	Only the primary
	====All servers in the replica set
	Only the secondaries
	A sum of all servers in the replica set
	The primary taken alone, plus a separate set with an average of all secondaries
#--------
Replica set state stransition
	blue to primary
	yellow to secondary
	red restart
# -------
opcounter-repls  - state transitions... on secondary inserts spikes.
repl lag -
replication oplog window - oplog how log secondary can fall to catch up with primary. if too big catchup need to sync from scratch.
#--------
replication oplog window - starts low and linear increase - oplog is not filling up, oplog enourmous. massiwe writes big increase and steady decrease.
#--------
opcounters

single delete, db.coll.remove() (1 delete op will explode in replica set for 35000 docs that many ops in other nodes too ) 
Extrapolating from this lesson, which of the following might cause different members of a replica set to have different opcounter values? Check all that apply.
	====You are reading only from the primary.
	====You have read preference set to secondaryPreferred.
	====The primary will receive reads from each secondary as they read the oplog.
# --------
# MMS Cluster view
insert - represent primaries if selected
inserts on primary != replica ops on secondaries
# --------
# MMS Alerts
can crete Alerts
read blog post http://www.mongodb.com/blog/post/five-mms-monitoring-alerts-keep-your-mongodb-deployment-track?mkt_tok=3RkMMJWWfF9wsRovuKjNZKXonjHpfsX96O8rWqGxlMI%2F0ER3fOvrPUfGjI4ATMZhMK%2BTFAwTG5toziV8R7DEK81o3cYQXhjh
# ----------
# 5 basic criteria setting alerts
1. Absolute limit :
	replication lag - lag increase  if it is bigger than 120 seconds send alert. if it happens like clock then these alerts are noise so some important  things gets missed. depends how long oplog size (inhours example 30 hours)
		warning level -> 100s
		critical level -> 270s
2. Normal (normal begaviour) 120s
3. Worrying () 240s
4. Critical - 3600s (critical lag on oplog)
5. false positives - like noise
# ---------
# host recovering alerts
1. host recovering -> when host goes "recovering" state (secondaries does recovering)

Which of the following will generate a host down alert? Check all that apply.
	====A mongod process monitored by the agent is killed.
	The MMS agent is killed.
	====A secondary in a replica set goes down.
	====An arbiter in a replica set goes down.
	A replica set cannot elect a primary.

# ----------
# 3. connections alerts
normal level of connections
	normal -> look into graphs over time to determine normal level of connections
	worrying -> 3
	critical -> yes, there is absolute limit (ulimit or final limit is memory of machine,because connection - 1MB)
		recommended example:
			primary: 10000
			secondary: 500
			mongos: 200
# ------------
# 4. lock percentage alerts
global percentage. look into how behaves databases (60 and 85 percent) do generally on primary, because ussually it will be less on secondaries

# ------------
# 5. replica - about 30 hours size.
decide a common point about 12 hours. 
but if 16 hours oplog then secondary cant catch up.
Maintenance window is 8am. provision 24 hours of oplog.
if replica drops then alert
	>12hours 
	>16hours
# -------------
# MMS administration
group - key in agent
group sizes:
logical segregation
	3 environments: dev (1), QA(1), Prod(2) (have agents for each environment) so we have 4 agents, 3 groups, 3 keys
	prod = 4 projects so multiplies
~100 host = pagination
-may hit resource issues for single agent
rule of thumb: 50 hosts per group

Note about Group names:

2 types of user = administrator, read-only (have to be added to receive alerts)
Jira | MMS users
jira.mongodb.org - shared authentication

adding users
	2 ways - known login, invite by email or jira username
# ----------
# mongostat
why use it? 
	frequency (1 second), MMS cant catch at particular moment  
disadvantages
	command line use, on demand tool
	tough to read
	hard to see trends

ps aux |grep mongos
mongo
sh.status()
exit

mongostat --help
mongostat --discover
	faults if hits disc
	locked - percentage
	idx miss correlated to faults. 
	ar|aw - active read active writes
	conn - connections
	repl - states

mongotop (on mongod works only)
	shows per collection info mongod how much time spends

insert  query update delete getmore command flushes mapped  vsize    res faults  locked db idx miss %     qr|qw   ar|aw  netIn netOut  conn set repl       time 
    *0    202     *0     *0       0    16|0       0  1.23g  3.11g   823m      0  test:0.0%          0       0|0     0|0    12k    21k    14  s1  PRI   20:35:15 
    *0    213     *0     *0       0     1|0       0  1.23g  3.11g   823m      0  test:0.0%          0       0|0     0|0    11k    11k    14  s1  PRI   20:35:16 
    *0    234     *0     *0       0     3|0       0  1.23g  3.11g   823m      0  test:0.0%          0       0|0     0|0    13k    12k    14  s1  PRI   20:35:17 
    *0    242     *0     *0       1     1|0       0  1.23g  3.11g   823m      0  test:0.0%          0       0|0     1|0    13k    12k    14  s1  PRI   20:35:18 
    *0    226     *0     *0       0     3|0       0  1.23g  3.11g   823m      0  test:0.0%          0       0|0     0|0    13k    12k    14  s1  PRI   20:35:19 
    *0    227     *0     *0       0     1|0       0  1.23g  3.11g   823m      0  test:0.0%          0       0|0     1|0    12k    11k    14  s1  PRI   20:35:20 
    *0    240     *0     *0       0     3|0       0  1.23g  3.11g   823m      0  test:0.0%          0       0|0     1|0    13k    12k    14  s1  PRI   20:35:21 
    *0    213     *0     *0       0     9|0       0  1.23g  3.11g   823m      0  test:0.0%          0       0|0     0|0    12k    18k    14  s1  PRI   20:35:22 
    *0    192     *0     *0       1     3|0       0  1.23g  3.11g   823m      0  test:0.0%          0       0|0     0|0    11k    10k    14  s1  PRI   20:35:23 
    *0    216     *0     *0       0     1|0       0  1.23g  3.11g   823m      0  test:0.0%          0       0|0     0|0    12k    11k    14  s1  PRI   20:35:24 

Which of the following statements is true?
	Horizontal scaling would decrease the write lock percentage.
	The index is too big to fit in memory.
	====The data set is staying the same size.
	The data set is too big to fit in memory.
	This member is a secondary.

# ----------
# iostat  (from sysstat) to track if we have slow disk
iostat -xmt 1 (x print extended information, t with timestamp, m show into MB) runs every second

avgqu-sz average queue sizes (page faults)
%util see if disc is strugling so to see io problem
svctm unreliabile metric
await service time and queue time (may show overloaded disc)

# netstat about existing tcp connections
statistics is inportant
netstat -s
look into TCP part (problem that it is accumulated from everything in that machine)
	failed connection attempts
	segments retransmited - look for increaseas, so if lots of retransmits then packets loss or latency
	bad segments - network corruption, bad data comes 
	resets sent - shows problem

onece found then look into TcpExt
	fast retransmits - bad if increase
	time recovered form packet loss due to SACK
	connections aborted due to timeout and similar

what do you do when you see problems? contact admins, traceroute pings and so on in network


If a disk was highly utilized and struggling to keep up, which CPU-level statistic would you expect to see increased?
	%user
	====%iowait
	%steal
	%nice
	%idle 
# ----------
# db.serverStatus()
asserts - 
	user 5 - rate per second
dur - durrability. info about journal
extra_info - can show page faults

these stats go to MMS
and some arent 

ttl connections

it is ran once a minute.

serverStatus was very slow - maybe global lock or big load. check if it is very slow all the time

Which of the following is NOT true of db.serverStatus?
	It tells you opcounter time series
	It tells you the current number of connections
	It tells you total uptime
	It tells you total network I/O
	====It tells you the names of hidden replica set members.







