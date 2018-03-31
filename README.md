# elasticsearch-tuning

## Tip 1 Set Num-of-shards to Num-of-nodes

Shard is the foundation of ElasticSearch’s distribution capability. Every index is splitted into several shards (default 5) and are distributed across cluster nodes. But this capability does not come free. Since data being queried reside in all shards (this behaviour can be changed by routing), ElasticSearch has to run this query on every shard, fetch the result, and merge them, like a map-reduce process. So if there’re too many shards, more than the number of cluter nodes, the query will be executed more than once on the same node, and it’ll also impact the merge phase. On the other hand, too few shards will also reduce the performance, for not all nodes are being utilized.

Shards have two roles, primary shard and replica shard. Replica shard serves as a backup to the primary shard. When primary goes down, the replica takes its job. It also helps improving the search and get performance, for these requests can be executed on either primary or replica shard.

Since number_of_shards of an index cannot be changed after creation (while number_of_replicas can), one should choose this config wisely. Here are some suggestions:

How many nodes do you have, now and future? If you’re sure you’ll only have 3 nodes, set number of shards to 2 and replicas to 1, so there’ll be 4 shards across 3 nodes. If you’ll add some servers in the future, you can set number of shards to 3, so when the cluster grows to 5 nodes, there’ll be 6 distributed shards.
How big is your index? If it’s small, one shard with one replica will due.
How is the read and write frequency, respectively? If it’s search heavy, setup more relicas.

## Tip 2 Tuning Memory Usage
ElasticSearch and its backend Lucene are both Java application. There’re various memory tuning settings related to heap and native memory.

**Set Max Heap Size to Half of Total Memory**
Generally speaking, more heap memory leads to better performance. But in ElasticSearch’s case, Lucene also requires a lot of native memory (or off-heap memory), to store index segments and provide fast search performance. But it does not load the files by itself. Instead, it relies on the operating system to cache the segement files in memory.

Say we have 16G memory and set -Xmx to 8G, it doesn’t mean the remaining 8G is wasted. Except for the memory OS preserves for itself, it will cache the frequently accessed disk files in memory automatically, which results in a huge performance gain.

Do not set heap size over 32G though, even you have more than 64G memory. The reason is described in this link.

Also, you should probably set -Xms to 8G as well, to avoid the overhead of heap memory growth.

**Disable Swapping**
Swapping is a way to move unused program code and data to disk so as to provide more space for running applications and file caching. It also provides a buffer for the system to recover from memory exhaustion. But for critical application like ElasticSearch, being swapped is definitely a performance killer.

There’re several ways to disable swapping, and our choice is setting bootstrap.mlockall to true. This tells ElasticSearch to lock its memory space in RAM so that OS will not swap it out. One can confirm this setting via http://localhost:9200/_nodes/process?pretty.

If ElasticSearch is not started as root (and it probably shouldn’t), this setting may not take effect. For Ubuntu server, one needs to add <user> hard memlock unlimited to /etc/security/limits.conf, and run ulimit -l unlimited before starting ElasticSearch process.

**Increase mmap Counts**
ElasticSearch uses memory mapped files, and the default mmap counts is low. Add vm.max_map_count=262144 to /etc/sysctl.conf, run sysctl -p /etc/sysctl.conf as root, and then restart ElasticSearch.

## Tip 3 Setup a Cluster with Unicast
ElasticSearch has two options to form a cluster, multicast and unicast. The former is suitable when you have a large group of servers and a well configured network. But we found unicast more concise and less error-prone.

Here’s an example of using unicast:

```
node.name: "NODE-1"
discovery.zen.ping.multicast.enabled: false
discovery.zen.ping.unicast.hosts: ["node-1.example.com", "node-2.example.com", "node-3.example.com"]
discovery.zen.minimum_master_nodes: 2
```

The discovery.zen.minimum_master_nodes setting is a way to prevent split-brain symptom, i.e. more than one node thinks itself the master of the cluster. And for this setting to work, you should have an odd number of nodes, and set this config to ceil(num_of_nodes / 2). In the above cluster, you can lose at most one node. It’s much like a quorum in Zookeeper.

## Tip 4 Disable Unnecessary Features
ElasticSearch is a full-featured search engine, but you should always tailor it to your own needs. Here’s a brief list:

Use corrent index type. There’re index, not_analyzed, and no. If you don’t need to search the field, set it to no; if you only search for full match, use not_analyzed.
For search-only fields, set store to false.
Disable _all field, if you always know which field to search.
Disable _source fields, if documents are big and you don’t need the update capability.
If you have a document key, set this field in _id - path, instead of index the field twice.
Set index.refresh_interval to a larger number (default 1s), if you don’t need near-realtime search. It’s also an important option in bulk-load operation described below.

## Tip 5 Use Bulk Operations
Bulk is cheaper

Bulk Read
Use Multi Get to retrieve multiple documents by a list of ids.
Use Scroll to search a large number of documents.
Use MultiSearch api to run search requests in parallel.

Bulk Write
Use Bulk API to index, update, delete multiple documents.
Alter index aliases simultaneously.

Bulk Load: when initially building a large index, do the following,
Set number_of_relicas to 0, so no relicas will be created;
Set index.refresh_interval to -1, disabling nrt search;
Bulk build the documents;
Call optimize on the index, so newly built docs are available for search;
Reset replicas and refresh interval, let ES cluster recover to green.

## Miscellaneous
File descriptors: system default is too small for ES, set it to 64K will be OK. If ulimit -n 64000 does not work, you need to add <user> hard nofile 64000 to /etc/security/limits.conf, just like the memlock setting mentioned above.
When using ES client library, it will create a lot of worker threads according to the number of processors. Sometimes it’s not necessary. This behaviour can be changed by setting processors to a lower value like 2:

```
val settings = ImmutableSettings.settingsBuilder()
    .put("cluster.name", "elasticsearch")
    .put("processors", 2)
    .build()
val uri = ElasticsearchClientUri("elasticsearch://127.0.0.1:9300")
ElasticClient.remote(settings, uri)
```

# QUICK WAY TO IMPROVE ELASTICSEARCH PERFORMANCE ON A SINGLE MACHINE
In Elasticsearch by Aurimas MikalauskasMay 24, 20152 Comments

It’s hard to find a server that has less than 4 cores and at least 2 disks these days. Multi-core, multi-disk environments have now become a commodity, yet not all software is built (or, in some cases, configured out of the box) to make the best use of that. Moreover, measuring and instrumenting such systems properly has become increasingly complex, which is why this has been one of the most interesting topics that I give talks in various tech conferences on.

Elasticserach is not an exception. Yes, it is multi-threaded and it does make a pretty good use of available CPUs or disks, especially at high concurrency environment. But. If you want your queries to return faster using as much CPU or Disk capacity as they possibly can, there’s something you can do about it.

## ELASTICSEARCH PERFORMANCE
First, I want to make it clear what do I mean with Performance here, because performance means different things for different people and in different contexts.

In this specific case, I want to focus on response time. Specifically, on a response time of a single search request. And if for the duration of the request, machine uses more resources – CPU, or disk – that is fine. As long as the search request returns faster.

Because we are doing this on a single machine, the process of such performance optimization is often called Scale-Up and that’s how we’re going to call it here too.

## WORKLOAD: CPU OR DISK BOUND?
It is important to talk about Disks and CPUs in this context because the type of workload you have, as well as the number of CPU cores and Disks on the system will determine how much you can Scale Up.

If your working set (amount of data, that is used to answer most search queries) fits in memory, then it is the number of CPU cores that will be limiting the response time of your query eventually.

If, on the other hand, queries will be reading from disk a lot, then you will end up with disk-bound workload.

I won’t go into measuring the workload in this article (leave a comment if that’s of an interest to you), but essentially if any given index is bigger than the available amount of memory on the system, then the performance will be limited by the number of disks you have on the server.

## THE SECRET TO BETTER PERFORMANCE
So here’s how Elasticsearch executes queries on the high levl.

First, it checks how many shards you have in your cluster (yes, on a single machine, it’s going to be a one node cluster), sends the very same query to all of the shards in parallel, then retrieves the results, aggregates them and sends back to the client.

Key phrase here is sends the query to all of the shards in parallel, because that’s the main area we can get some performance improvements. As you may have guessed already, the number of shards for a given index will determine how many parallel requests server will run internally while searching.

By default, 5 shards are created per index. Which means that if you have a 24 core system, you will only use 5 cores for any single search query in a CPU bound workload. Likewise, if you have a 50-disk RAID, or a fast SSD that supports parallel I/O requests (all flash disks support parallel requests), with default index configuration you will not be making the best use of that powerful hardware.

In other words, all we want to change is the number of shards. But…

## HOUSTON, WE HAVE A PROBLEM
All would be nice, but the problem is that in current version of Elasticsearch, you can’t change the number of shards for existing indexes. You can only set it during the index creation time. Therefore, you will have to reindex your data with the new number of shards. Reindexing is nicely described in The Definitive Guide.

So you want to create an index with the appropriate number of shards (say, 24):

```
PUT /the_new_index
{
    "settings": {
        "number_of_shards" :   24,
        "number_of_replicas" : 0
    }
}


PUT /the_new_index
{
    "settings": {
        "number_of_shards" :   24,
        "number_of_replicas" : 0
    }
}
```

And then load all your data into this new index.

## HOW MANY SHARDS?
While I have not done any benchmarks to determine the perfect ratio between number of shards and number of CPU cores or disks, performance wise, it’s better to have more shards than fewer because the OS will schedule the overlapping requests properly.

But as a rule of thumb, I would align them either to the number of CPU cores or to the number of disks, depending on the workload.

Oh and if you would like to have some benchmarks done, do let me know in comments.

## WILL THIS HELP ALL THE TIME?
No. A degree to which it will improve the performance will depend mostly on how much data your search requests will match. If they match hundreds of thousands of documents, then the query will be spending a lot more time in the aggregation phase which is single-threaded for the most part and therefore this optimization will be less helpful.

If, on the other hand, your search requests are highly selective, then this is your ticket to 2, 3 or even up to 5 times faster search queries if you have a lot of cores and/or disks.

# don’t use millisecond timestamps in Elasticsearch if seconds will do!

# https://www.ebayinc.com/stories/blogs/tech/elasticsearch-performance-tuning-practice-at-ebay/

# https://www.elastic.co/blog/found-elasticsearch-in-production

# https://www.loggly.com/blog/nine-tips-configuring-elasticsearch-for-high-performance/

# https://www.elastic.co/blog/found-optimizing-elasticsearch-searches

# 
