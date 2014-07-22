Homework 1.1
----
Following the instructions in an earlier lesson, install the MMS monitoring agent. Text instructions for installing the agent are [here](http://mms.mongodb.com/help/monitoring/tutorial/set-up-mms/).
This is easy.

Homework 1.2
----
You must complete Homework 1.1 before doing this problem.

Click the "Initialize" button in MongoProc to begin generating data for MMS. Clicking "Initialize" will cause a script to run that will insert and delete documents in the homework_1_2 collection of the m202 database (on the host you set up for Homework 1.1). Let this run for approximately 2 hours.

After 2 hours, look at your MMS Monitoring console and determine how many inserts per second there are for your host. Please use a resolution of 1 hour. At higher resolutions the data will jump around too much for you to see the average value.

When there is enough data for you to determine an answer, insert a document into the answers collection of the m202 database. The document you insert should have the following form:

```json
{ "homework" : "homework_1.2", "answer" : 12313 }
```
The value for the answer field in this document should be the number of inserts per second for your host. Please be aware that your answer may not be the same as that for other students. Validation for this problem accounts for such differences.

The answer will be just from 1 hour. Take integer part of float and insert into answers collection

Homework 1.3
----
```bash
#install munin-node
sudo apt-get install munin-node
#see if running
ps -ef | grep "munin"
#if not, then start it
/etc/init.d/munin-node start #sudo service munin-node start
# check if you can connect
telnet [HOSTNAME] 4949
fetch iostat
fetch iostat_ios
fetch cpu

# if can't connect, check logs
# You may need to add allowed host to connect to this machine
nano /etc/munin/munin-node.conf
# add
allow 10.202.30.210
```

Homework 1.4
----
![backflush](/week1/background_flush.png)
![lock](/week1/lock.png)
![btree](/week1/btree.png)
![pagefaults](/week1/page_faults.png)

Which of the conditions below are implied by these graphs?

1.  **The server is under heavy write load**
2.  An index was removed shortly before 12:30
3.  **Queries accessed data not in memory starting shortly before 12:30**
4.  The server activity is constant over the period viewed

Explanations:

1.  A write-heavy database might regularly see >60% lock [source](http://blog.mms.mongodb.com/post/78650784046/learn-about-lock-percentage-concurrency-in-mongodb). Also background flush avg is high
2.  High page faults would indicate the database or index does not fit in ram. Btree access indicates access to index. Dropping an index under load, or adding an index would spike the btree metric. So btree graph does not have sudden spikes thus third aswer is correct
3.  It is seen in page faults graph
4.  You can see heavy write load and some page faults so it is not constant

