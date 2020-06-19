# Improve the Performance of Apache Geode Persistence Recovery

## Introduction

Apache Geode is a data management platform that provides real-time, consistent access to data-intensive applications throughout widely distributed cloud architectures.
Geode pools memory, CPU, network resources, and optionally local disk across multiple processes to manage application objects and behavior. 
It uses dynamic replication and data partitioning techniques to implement high availability, improved performance, scalability, and fault tolerance. 
In addition to being a distributed data container, Apache Geode is an in-memory data management system that provides reliable asynchronous event notifications and guaranteed message delivery.

Apache Geode offers super fast write-ahead-logging (WAL) persistence with a shared-nothing architecture that is optimized for fast parallel recovery of nodes or an entire cluster.
Persistence provides a disk backup of region entry data. The keys and values of all entries are saved to disk, like having a replica of the region on disk. 
When the member stops for any reason, the region data on disk remains.  The disk data can be used at member startup to populate the same region.
Each member has its own set of disk stores, and they are completely separate from the disk stores of any other member.

Disk store files include store management files, access control files, and the operation log, or oplog, files, consisting of one file for deletions and another for all other operations.
For example, in the file system, there are oplog files with crf, drf and krf file extensions. 
crf	oplog contains create, update, and invalidate operations for region entries. 
drf	oplog contains delete operations. 
And krf oplog contains the key names as well as the offset for the value within the crf file.
The krf oplog will improve startup performance by allowing Geode to load the entry values in the background after the entry keys are loaded.
When you start a member with a persistent region, the data is retrieved from disk stores to recreate the member’s persistent region. 
Entry keys are loaded from the key file in the disk store before considering entry values. Once all keys are loaded, Geode loads the entry values asynchronously.

## Challenges for the Performance Persistence Recovery

In the recent tests on the cloud environment, we have observed that the persistence recovery takes a long time. 
We have a Geode cluster with one locator and four servers. We shutdown all the servers, but keep the locators running. And then restart all the servers together. 
We notice that two of the servers restarted quickly. The other two servers takes significantly longer time to recover the persisted data.

## Remove Unnecessary Thread Synchronization

In the Geode server logs, the ThreadsMonitor has put warning messages in the logs to tell which thread is stuck for how long, and which thread currently hold the lock together with the thread stack. 
The logs show that the thread is waiting on the lock on a HashMap. The log also shows that the initialization of a region takes unusually long time. All these clues are correlated. 
With the help of the warning messages and the stack trace, we have identified that in the source code, the synchronization of HashMap has costed the persistence recovery to slow down significantly. 
One thread on one server is holding the lock while recovering the persisted krf oplogs. 
While the other thread is waiting for the lock to be released before replying a message to the other server, which caused the other server to be blocked before its persistence recovery. 
And the blocked server has to wait until the server who holds the lock on HashMap finishes persistence recovery. 
The thread synchronization on one server has affected the progress of the other server. In Geode, the servers exchange messages to collaborate on certain work like region creation.
For log analysis details, please refer to GEODE-7945 and its pull request.

Once we understand the root cause, the solution becomes clear. We'd better replace HashMap with ConcurrentHashMap, to remove unnecessary synchronization among the threads.
With ConcurrentHashMap, the persistence recovery time has reduced by 30% for recovering the same amount of persistent data during cluster restart.

## Parallel Disk stores Recovery

When further analyzing the logs, we noticed that the disk stores are recovered sequentially with single thread. If the disk stores are recovered in parallel, the persistence recovery performance can be dramatically improved. 
When each region are on different disk store, parallel disk store recovery makes parallel region recovery possible. 
Especially when the disk stores are on different disk controllers. So that the disk stores don't have to compete for the same disk controller. 
With the completion of GEODE-8045, parallel disk store recovery is introduced. 
For example, when recovering two regions on two separate disk stores, we can reduce the persistent recovery time in half, compared to the case that two regions sharing the same disk store. 
With more disk stores, the performance of persistence recovery can be further improved from parallel disk store recovery.

## Conclusion

Geode shared-nothing persistence architecture is powerful for fast parallel recovery of nodes or an entire cluster. 
With recent performance improvement, we further removed the unnecessary thread synchronization during persistence recovery.
And we also introduced parallel disk store recovery within each node. The improvement has made Geode parallel recovery even faster. 

## Interested Apache Geode? Try it!

Apache Geode Home Page: https://geode.apache.org/

Apache Geode Docker Image: https://hub.docker.com/r/apachegeode/geode/

Apache Geode Examples: https://github.com/apache/geode-examples
