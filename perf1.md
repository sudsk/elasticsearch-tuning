The Loggly service utilizes Elasticsearch (ES) as the search engine underneath a lot of our core functionality. As Jon Gifford explained in his recent post on Elasticsearch vs Solr, log management imposes some tough requirements on search technology. To boil it down, it must be able to:

Reliably perform near real-time indexing at huge scale – in our case, more than 100,000 log events per second
Simultaneously handle high search volumes on the same index with solid performance and efficiency
When we were building our Gen2 log management service, we wanted to be sure that we were setting all configurations in the way that would optimize Elasticsearch performance for both indexing and search. Unfortunately, we found it very difficult to find this information in the Elasticsearch documentation because it’s not located in one place. This post summarizes our learnings and can serve as a checklist of configuration properties you can reference to optimize Elasticsearch (also referred to herein as ES) for your application.

Note: These tips were last updated in September 2016. Some of the comments below may reference older tips.

Tip #1: Planning for Elasticsearch index, shard, and cluster state growth: biggest factor on management overhead is cluster state size.
Loggly - Tweet This
ES makes it very easy to create a lot of indices and lots and lots of shards, but it’s important to understand that each index and shard comes at a cost. If you have too many indices or shards, the management load alone can degrade your ES cluster performance, potentially to the point of making it unusable. We’re focusing on management load here, but running too many indices/shards can also have pretty significant impacts on your indexing and search performance.

The biggest factor we’ve found to impact management overhead is the size of the Cluster State, which contains all of the mappings for every index in the cluster. At one point, we had a single cluster with a Cluster State size of over 900MB! The cluster was alive but not usable.

Let’s walk through some numbers so you can get a feel for what can happen…

Imagine you have an index that has 50k of mappings (for us, that’s about 700 fields). If you have an index per hour, then you’re adding 24 x 50k of cluster state per day, or 1.2MB. If you have a year’s worth of data in your system, then you’re at 438MB of cluster state (and 8760 indices, 43800 shards). If we compare this to an index a day (18.25MB, 365 indices, 1825 shards), you can see that hourly indices put you in a different league.

The nice thing is that it is actually pretty easy to do these projections, once you have some realistic data in your system. You should be able to see how much cluster state and how many indices/shards your cluster will have to deal with. You really should do this exercise before you go into production, to avoid that 3:00 a..m.. call that the cluster is misbehaving.

In terms of configuration, you have complete control over how many indices you have in your system (and how many shards they have), which should let you stay well away from the danger zone.

Tip #2: Know your Elasticsearch cluster topology before you set configs
Loggly is running ES with separate master and data nodes. We won’t be going into too much detail about that right now (look out for a subsequent post), other than to say that you need to determine your deployment topology in order to make the right Elasticsearch configuration decisions.

In addition, we use separate ES client nodes for both indexing and searching. This takes some load off the data nodes, but more importantly means that our pipeline can talk to a local client, which then communicates with the rest of the cluster.

You establish your ES nodes as data and master using two properties that are set as true or false. These are:

Master node: node.master:true node.data:false 
Data node: node.master:false node.data:true 
Client node: node.master:false node.data:false
So that was the easy part. Now we’ll talk about some advanced ES properties that deserve your attention. Their default settings are sufficient for most deployments, but if your ES use cases are anywhere near as tough as those we see with log management, you’ll get a lot of benefit from the advice below.

Tip #3: mlockall offers the biggest bang for the Elasticsearch performance efficiency buck
Loggly - Tweet This
Linux divides its physical RAM into chunks of memory called pages. Swapping is the process whereby a page of memory is copied to the preconfigured space on the hard disk, called swap space, to free up that page of memory. The combined sizes of the physical memory and the swap space is the amount of virtual memory available.

Swapping does have a downside. Compared to memory, disks are very slow. Memory speeds can be measured in nanoseconds, while disks are measured in milliseconds; so accessing the disk can be tens of thousands times slower than accessing physical memory. The more swapping that occurs, the slower your process will be, so you should avoid swapping at all cost.

The mlockall property in ES allows the ES node not to swap its memory. (Note that it is available only for Linux/Unix systems.) This property can be set in the yaml file by doing the following.

bootstrap.mlockall: true

In the 5.x releases, this has changed to bootstrap.memory_lock: true.

mlockall is set to false by default, meaning that the ES node will allow swapping. Once you add this value to the property file, you need to restart your ES node. You can verify if the value is set properly or not by doing the following:

curl http://localhost:9200/_nodes/process?pretty

if you are setting this property, make sure you are giving enough memory to the ES node using the -DXmx option or ES_HEAP_SIZE.

Tip #4: discovery.zen properties control the discovery protocol for Elasticsearch
Loggly - Tweet This
Zen discovery is the default mechanism used by Elasticsearch to discover and communicate between the nodes in the cluster. Other discovery mechanisms exist for Azure, EC2 and GCE. Zen discovery is controlled by the discovery.zen.* properties.

In 0.x and 1.x releases both unicast and multicast are available, and multicast is the default. To use unicast with these versions of ES, you need to set discovery.zen.ping.multicast.enabled to false.

From 2.0 onwards unicast is the only option available for Zen discovery.

To start with, you must specify the group of hosts that are used to communicate for discovery, using the property discovery.zen.ping.unicast.hosts. To keep things simple, use the same value for this property on all hosts in your cluster. We define this list using the names of our master nodes.

The discovery.zen.minimum_master_nodes control the minimum number of eligible master nodes that a node should “see” in order to operate within the cluster. It’s recommended that you set it to a higher value than 1 when running more than 2 nodes in the cluster.  One way to calculate value for this will be N/2 + 1 where N is number of master nodes.

Data and master nodes detect each other in two different ways:

By the master pinging all other nodes in the cluster and to verify they are up and running
By all other nodes pinging the master nodes to verify if they are up and running or if an election process needs to be initiated
The node detection process is controlled by discover.zen.fd.ping_timeout property. The default value is 30s, which determines how long the node will wait for a response. This property should be adjusted if you are operating on a slow or congested network. If you are on slow network, set the value higher. The higher the value, the smaller the chance of discovery failure.

Loggly has configured our discovery.zen properties as follows:

discovery.zen.fd.ping_timeout: 30s
discovery.zen.minimum_master_nodes: 2
discovery.zen.ping.unicast.hosts: ["esmaster01","esmaster02","esmaster03"]
The above properties say that node detection should happen within 30 seconds; this is done by setting discovery.zen.fd.ping_timeout. In addition, two minimum master nodes should be detected by other nodes (we have 3 masters). Our unicast hosts are esmaster01, esmaster02, esmaster03.

Tip #5: Watch out for DELETE_all!
Loggly - Tweet This
It’s really important to know that the DELETE API in ES allows you to delete multiple indices with a single request, using wildcards, or even ALL of your indices using _all as the index name. For example:

curl -XDELETE 'http://localhost:9200/*/'

This power is very useful, but also very dangerous, especially in a production environment. In all of our clusters, we disable it using the action.destructive_requires_name:true setting.

This setting was introduced in 1.0, and replaced the action.disable_delete_all_indices setting used in 0.90.

Tip #6: Use Doc Values
Doc Values are used by default in 2.0 and above, but you have to explicitly enable them in earlier versions of ES. They provide significant advantages over “normal” fields when you’re doing a lot of sorting or aggregations, at a slight cost in indexing and disk space. Essentially, they turn ES into a columnar store, and make many of the analytical features of ES far more performant than you might expect.

To understand why, we can compare Doc Values with “normal” fields in ES.

When you use a normal field for sorting or aggregations, it is loaded into the fielddata cache. The first time a field is cached, ES has to allocate space on its heap large enough to hold every value, then fill that with the value from every document. This process can take some time, since it will probably have to read those values from disk. Once this is done, any use of that field will use this cached data, and will be fast. If you try to fit too many fields into this cache, it will evict some fields, and subsequent use of those fields will force them to be loaded again, with the same start-up cost. To be most effective, you want to minimize or eliminate evictions, which means you’re limited in the number of fields you can cache in this way.

By contrast, Doc Values fields use a disk-based data structure that can be memory mapped into the process space, and thus has no impact on heap usage, but provides essentially the same performance as the fielddata cache. There is still a small start-up cost when these fields are first used as the data is read from disk, but this is being handled by your OS filesystem cache so only the data you need is actually read.

Doc Values, therefore, minimize heap usage (hence garbage collections), and can take advantage of your OS filesystem cache to minimize disk reads.

Tip #7: Navigating Elasticsearch’s allocation-related properties 
Loggly - Tweet This
Shard allocation is the process of allocating shards to nodes. This can happen during initial recovery, replica allocation, or rebalancing. Or it can happen when handling nodes that are being added or removed.

The cluster.routing.allocation.cluster_concurrent_rebalance property determines the number of shards allowed for concurrent rebalance. This property needs to be set appropriately depending on the hardware being used, for example the number of CPUs, IO capacity, etc. If this property is not set appropriately, it can impact the performance of ES indexing.

cluster.routing.allocation.cluster_concurrent_rebalance:2

By default the value is set at 2, meaning that at any point in time only 2 shards are allowed to be moving. It is good to set this property low so that the rebalance of shards is throttled and doesn’t affect indexing.

The other shard allocation property is cluster.routing.allocation.disk.threshold_enabled. If this property is set to true (the default), the shard allocation will take free disk space into account while allocating shards to a node. Turning this off can result in ES allocating shards to a node which doesn’t have enough available disk to account for that shards growth.

When enabled, the shard allocation takes two watermark properties into account: low and high.

The low watermark defines the disk usage point beyond which ES won’t allocate new shards to that node. (default is 85%)
The high watermark defines the disk usage point beyond which the shards will start moving off the node (default is 90%)
Both of these can be defined as either a percentage of disk used (e.g. “80%” means 80% of the disk has been used, or 20% is free), or as a minimum size of disk available (e.g. “20GB” means there is 20GB of free disk available to the node)

The defaults are fairly conservative if you have a lot of small shards. For example, if you have a 1TB drive, and your shards are typically 10GB in size, then in theory you could put 100 shards on that node. With the default settings, you’ll only be able to put 80 of those shards on the node before ES decides it is full.

To work out what value you should use, you should look at how big your shards are going to get over their lifetime, and work backwards from there, being sure to include a safety factor. In the example above, we might only have 5 shards being written to, so need to make sure we have 50GB available at all times. For a 1TB drive, this translates to a 95% low water mark, with no safety factor. Adding, say, a 50% safety factor, means we should make sure to have 75GB free, or a 92.5% low water mark.

Tip #8: Recovery properties allow for faster restart times
ES includes several recovery properties which improve both Elasticsearch cluster recovery and restart times. The value that will work best for you depends on the hardware you have in use (disk and network being the usual bottlenecks), and the best advice we can give is to test, test, and test again.

To control how many shards can be simultaneously in recovery on a single node, use:

cluster.routing.allocation.node_concurrent_recoveries

Recovering shards is a very IO-intensive operation, so you should adjust this value with real caution. In 5.x releases, this is split into:

cluster.routing.allocation.node_concurrent_incoming_recoveries

cluster.routing.allocation.node_concurrent_outgoing_recoveries

To control the number of primary shards initialized concurrently on a single node, use:

cluster.routing.allocation.node_initial_primaries_recoveries

To control the number of parallel streams to open to support recovery of a shard:

indices.recovery.concurrent_streams

Closely tied to the number of streams, is the total network bandwidth available for recovery:

indices.recovery.max_bytes_per_sec

With all of these properties, the best values will depend on the hardware you’re using. If you have SSDs and a fast (10G) ethernet fabric, then the values that work best may be very different than if you’re using spinning hard drives and 1G ethernet.

All of the properties described above get used only when the cluster is restarted.

Tip #9: Threadpool properties simplifies indexing, prevents data loss 
Loggly - Tweet This
Elasticsearch node has several thread pools in order to improve how threads are managed within a node.

At Loggly, we use _bulk requests for indexing, and we have found that setting the right value for bulk thread pool using the threadpool.bulk.queue_size property is crucial in order to avoid _bulk retries, and thus potential data loss.

threadpool.bulk.queue_size: 5000

This tells ES the number of shard requests that can be queued for execution in the node when there is no thread available to execute a bulk request. This value should be set according to your bulk request load. If your bulk request number goes higher than queue size, you will get a RemoteTransportException as shown below.

As we said above, the bulk requests queue contains one item per shard, so this number needs to be higher than the number of concurrent bulk requests you want to send multiplied by the number of shards in those requests. For example, a single bulk request may contain data for 10 shards, so even if you only send one bulk request, you must have a queue size of at least 10. Setting this value “too high” will chew up heap in your JVM (and indicates that you’re pushing more data into your cluster than it can comfortably index), but does let you hand off some queuing to ES which simplifies your clients.

You either need to keep the property value higher than your accepted load or gracefully handle RemoteTransportException in your client code. If you don’t handle the exception, you will  end up losing data. We simulated the exception shown below by sending more than 10 bulk requests with a queue size of 10.

RemoteTransportException[[<Bantam>][inet[/192.168.76.1:9300]][bulk/shard]]; nested: EsRejectedExecutionException[rejected execution (queue capacity 10) on org.elasticsearch.action.support.replication.TransportShardReplicationOperationAction$AsyncShardOperationAction$1@13fe9be];
Bonus Tip for Pre-2.0 Users: Minimize Mapping Refreshes
If you’re still using a pre-2.0 version of ES and have frequent changes to your field mappings, you may find that your cluster has a large number of refresh_mappings requests in your pending tasks queue. In itself, this is not so bad, but it can snowball and impact the performance of your cluster.

If you do see this behavior, there is an Elasticsearch configuration parameter that can help. We use this parameter like so:

indices.cluster.send_refresh_mapping: false

So, what is this, and why does it work?

When a new field is detected as part of indexing, the data node it is being added to updates its mapping, and sends that new mapping to the master. If this new mapping is still in the Masters pending task queue when the Master sends out its next cluster state, then the data node will be receiving an “old” version of the mapping. Normally, this would cause it to send a refresh mapping request to the master, since as far as the data node is concerned, the Master has the wrong mappings. This is sensible default behavior – the node should do something to ensure the Master has the right mappings, and resending the new mapping is the right way to do this.

However, if there are a lot of mapping changes happening, and the Master can’t keep up, there is a stampeding horde effect, and the data node can flood the Master with refresh messages.

The indices.cluster.send_refresh_mapping parameter allows us to disable the default behavior, thus eliminating these refresh_mapping requests going from data node to Master, which allows the Master to stay up to date. Even without the refresh requests, the Master will eventually see the original mapping change, and send out an updated cluster state including that change.

In Summary: Optimizing Elasticsearch configuration properties is the key to its elasticity
The depth of configuration properties available in Elasticsearch has been a huge benefit to Loggly since our use cases take Elasticsearch to the edge of its design parameters (and sometimes beyond). If the ES default configurations are working perfectly adequately for you in the current state of your application’s evolution, rest assured that you’ll have plenty of levers available to you as your application grows.
