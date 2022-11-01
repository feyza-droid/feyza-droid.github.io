---
layout: post
title:  MapReduce
date:   2022-11-01
description: MapReduce 
tags: software-engineering
categories: coding
---

## Distributed File Systems

<br>

### Mining of Massive Datasets

<br>

### Map-Reduce
- Distributed File System
- Computational Model
- Scheduling and Data Flow
- Refinements

What happens when the data is too big to fit in the memory? In classical data mining, you bring portion of the data, process it in batches and write back results to disk. With the 2016 technology, it takes 46+ days just to read 10 billion web pages. This is not feasible/unacceptable, therefore you need to parallize it by using multiple disks and CPUs. The main aim with cluster computing is to reduce the time spent. 

<br>

### Cluster Architecture

You have the racks consisting of commodity Linux nodes. You use them because they are very cheap. Each rack has 16 to 64 of these commodity Linux nodes. These nodes are connected by a switch. This switch in a rack is typically a gigabit switch so there is 1 Gbps bandwidth between any pair of nodes in rack. The racks themselves are connected by backbone switches. This backbone switches have higher bandwidths 2 to 10 Gbps between racks. You rack up multiple racks and you get a data center. This is the standard classical architecture for storing and mining very large datasets. In 2011, unofficial sources estimated that Google had 1 million nodes like this.

<br>

### Cluster Computing Challenges

1 - Node failures
    - A single server can stay up for 3 years (1000 years)
    - 1000 servers in cluster -> 1 failure/day
    - 1M servers in cluser -> 1000 failures/day

- [Persistency] How to store data persistently and keep it available if nodes can fail? Persistence means that once you store the data, you're guaranteed you can read it again. But if the node in which you stored the data fails, then you can't read the data. You might even lose the data. 

- [Availability] How to deal with node failures during a long-running computation? (Availability.) When you run half way through the computation, at this critical point, a couple of nodes fail. And that node had data that is necessary for the computation. In the worst case, you may have to go back and restart the computation all over again. But when you are running this computation the node again may fail. Therefore you need an infrastructure that can hide these kind of node failures and let the computation go to completion even if nodes fail.

2 - Network bottleneck
    - Network bandwidth = 1 Gbps
    - Moving 10TB takes approximately 1 day

- You need a framework that doesn't move data around so much while it's doing computation.

3 - Distributed programming is hard!
    - Need a simple model that hides most of the complexity of distributed programming. And makes it easy to write algorithms that can mine very massive datasets.
    - Hard to write a program so that to avoid race conditions and various kind of complications. 

<br>

### Map-Reduce

- Map reduce addresses the challenges of cluster computing
    - [To solve 1st problem about Persistency and Availability] Store data redundantly on multiple nodes for persistence and availability (Redundant Storage Infrastructure)
    - [To solve 2nd problem about Network Bottleneck] Move computation close to data to minimize data movement
    - [To solve 3rd problem about Distributed Programming is hard] Simple programming model to hide the complexity of all this magic

<br>

### Redundant Storage Infrastructure

- Distributed File System
    - It is a file system that stores data across a cluster, but stores each piece of data multiple times.
    - It provides global file namespace, redundancy and availability
    - There are multiple implementations of distributed file systems. E.g. Google GFS (Google file system); Hadoop HDFS (Hadoop distributed file system)

- Typical usage pattern
    - There are some typical usage patterns that these distributed file systems are optimized for.
    - Once the data is written, it is read very often. But when it is updated, it's updated through appends.
    - Typical usage pattern consists of writing the data once, reading it multiple times and appending to it occasionally.
    - Huge files (100s of GB to TB)
    - Data is rarely updated in place
    - Reads and appends are common

- Data kept in "chunks" spread across machines (Example, one file is divided into 6 chunks and kept in different servers)
- Each chunk replicated on different machines (ensures persistence and availability). Replicas of the chunk are never on the same chunk server. They are always on different chunk servers. 
- [Bring computation to data!] Chunk servers also serve as compute servers. Whenever your computation has to access data, that computation is actually scheduled on the chunk server that actually contains the data. This way you avoid moving data to where the computation needs to run, but instead you move the computation to where the data is. That is how you avoid unnecessary data movement. 

- Chunk servers
    - File is split into contiguous chunks (16-64 MB)
    - Each chunk replicated (usually 2x or 3x). 3x replication is the most common
    - Try to keep replicas in different racks (when the switch on a rack fails, the entire rack becomes inaccessible)

- Master node
    - a.k.a. Name node in Hadoop's HDFS
    - Stores metadata about where files are stored (For example, it'll know that file one is divided into 6 chunks and locations of each of the 6 chunks and the locations of the replicas)
    - Might be replicated (Otherwise it will become a single point of failure)

- Client library for file access
    - When a client or an algorithm that needs to access the data tries to access a file it goes through the client library. 
    - Talks to master to find chunk servers that actually store the chunks
    - After talking to master and learning the chunk servers location, client directly connects to chunk servers to access data
    - So the data access actually happens in peer-to-peer fashion without going through the master node

<br>

<hr>

## The MapReduce Computational Model

### Programming Model: MapReduce

<br>

### Warm-up Task:

    - We have a huge text document
    - Count the number of times each distinct word appears in the file
    - Sample applications
        - Analyze web server logs to find popular URLs
        - Term statistics for search engine

- Case 1:
    - File too large for memory, but all <word, count> pairs fit in memory
    - Simple approach by using HashTable would work (word (key) -> count) to keep track of the count of each unique word

- Case 2:
    - Even the <word, count> pairs don't fit in memory
    - words(doc.txt) | sort | uniq -c (where words takes a file and outputs the words in it, one per a line, uniq -c counts the occurrences of the same word)
    - case 2 captures the essence of MapReduce (great thing is that it is naturally parallelizable)

- Map [words(doc.txt)]
    - Scan input file record-at-a-time
    - Extract something you care about from each record (keys), in this case it is words
- Group by key [sort]
    - Sort and shuffle
- Reduce [uniq -c]
    - Aggregate, summarize, filter or transform
    - Write the result

Outline stays the same, Map and Reduce change to fit the problem

<br>

### More formally

- Input: a set of key-value pairs
- Programmer specifies two methods:
    - 1st METHOD - Map(k, v) -> <k', v'>
        - Takes a key-value pair and outputs a set of key-value pairs
        - There is one Map call for every (k,v) pair

    - 2nd METHOD - Reduce (k', <v'>) -> <k', v''>
        - All values v' with same key k' are reduced together
        - There is one Reduce function call per unique key k'

<br>

MapReduce system is built around doing only sequential reads of files and never random accesses there it is more efficient.

<div class="row justify-content-sm-center">
    <div class="col-sm-8 mt-3 mt-md-0">
        {% include figure.html path="assets/img/posts/map_reduce_word_counting.png" title="Map Reduce Word Counting" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>

<br>

```
map(key, value): # key: document name; value: text of the document
    for each word w in value: # it scans the input document, value here is the input document
        emit(w, 1) # output is the set of intermediate key-value pairs, for each word in the input document it emits that word and the number 1, it is a tuple whose key is the word and whose value is the number 1
```

```
reduce(key, values): # key: a word; value: an iterator over counts
    result = 0
    for each count v in values:
        result += v # sums all of the values (in this case sums all 1s to count the occurrences of the unique word)
    emit(key, result) # key is the word and value is the sum for this example
```

<br>

### Example: Host size

- Suppose we have a large web corpus with a metadata file formatted as follows:
    - Each record of the form: (URL, size, date, ...)
- For each host, find the total number of bytes

- Map
    - For each record, output (hostname(URL), size)
- Reduce
    - Sum the sizes for each host

<br>

### Example: Language Model

- Count number of times each 5-word sequence occurs in a large corpus of documents

- Map
    - Extract (5-word sequence, count) from document
- Reduce
    - Combine the counts

<hr>

## Scheduling and Data Flow

<div class="row justify-content-sm-center">
    <div class="col-sm-8 mt-3 mt-md-0">
        {% include figure.html path="assets/img/posts/map_reduce_a_diagram.png" title="Map Reduce A Diagram" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
    <div class="col-sm-8 mt-3 mt-md-0">
        {% include figure.html path="assets/img/posts/map_reduce_in_parallel.png" title="Map Reduce In Parallel" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>

### Map-Reduce: Environment

- Map-Reduce environment takes care of:
    - Partitioning the input data
    - Scheduling the program's execution across a set of machines
    - Performing the group by key step
    - Handling node failures
    - Managing required inter-machine communication

<br>

### Data Flow

- Input and final output are stored on the distributed file system (DFS):
    - Scheduler tries to schedule map tasks "close" to physical storage location of input data (The Map-Reduce system try to schedule each map task on a chunk server that holds a copy of the corresponding chunk. So there is no actual copy, a data copy associated with the map step of the Map-Reduce program.)
- Intermediate results are stored on local file system (FS) of Map and Reduce workers (Why are such debated results not stored in the distributed file system? Because there is some overhead to storing data in the distributed file system. To avoid more network traffic.)
- Output is often input to another MapReduce task

<br>

### Coordination: Master

- Master node takes care of coordination:
    - Task status: (idle, in-progress, completed)
    - Idle tasks get scheduled as workers become available
    - When a map task completes, it sends the master the location and sizes of its R intermediate files (on local file system), one for each reducer
    - Master pushes this info to reducers
    - Once the reducers know that all the mappers map tasks are completed, then they copy the intermediate file from each of the map tasks. And then they can proceed with their work.

- Master pings workers periodically to detect failures

<br>

### Dealing with failures

- Map worker failure
    - Map tasks are completed or in-progress at worker are reset to idle (The reason why we are resetting the completed tasks is that since the intermediate results are written to local file system of the map worker, intermediate results might be lost if the map worker fails. If the map worker fails, then the node fails. Then all the intermediate output created by all the map tasks that have ran on that worker, are lost.)
    - Idle tasks eventually rescheduled on other worker(s)

- Reduce worker failure
    - Only in-progress tasks are reset to idle (The reason why we aren't resetting the completed tasks is that since the output is written to distributed file system, the output is not lost even if the reduce worker fails. While completed tasks don't need to be redone.)
    - Idle Reduce tasks restarted on other workers(s)

- Master failure
    - MapReduce task is aborted and client is notified

<br>

### How many Map and Reduce jobs?

- M map tasks, R reduce tasks
- Rule of thumb:
    - Make M much larger than the number of nodes in the cluster
    - One DFS chunk per map is common (Rule of thumb is to have on map task per DFS chunk. The reason is simple. Imagine that there is one map task per node in the cluster and during processing the node fails. If a node fails then that map task needs to be rescheduled on another node in the cluster when it becomes available. Since all the other nodes are processing, one of those nodes has to complete before this map task can be scheduled on that node and so, the entire computation is slowed down by the time it takes to complete this map task to redo the failed map task. So instead of one map task on a given node, there are many small map tasks on a given node, and that node fails, then those map tasks can then be spread across all the available nodes and so the entire task will complete much faster.)
    - Improves dynamic load balancing and speeds up recovery from worker failures
- Usually R is smaller than M (and usually it is even smaller than the total number of nodes in the system)
    - Because output is spread across R files (It is usually convenient to have the output spread across a small number of nodes rather than across a large number of nodes)

<hr>

## NOTES
- This is my summary of the videos below.
- Be aware of that videos are recorded in 2016. Some information may be outdated.

<hr>

## References
    - https://www.youtube.com/watch?v=jDlrfBLAIuQ
    - https://www.youtube.com/watch?v=VEh91xTIEIU
    - https://www.youtube.com/watch?v=4_Eco-v4wlI