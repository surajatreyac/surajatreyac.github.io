---
layout: post
title:  "Elastically adding and removing nodes in Akka cluster"
date:   2015-08-03 11:00:00
categories: akka, distributed systems
---

In the previous post, we saw the internals of Akka cluster and how it communicates with each other using gossip protocol and heartbeat for failure detection. Also, I had promised that I would explain the Akka cluster with a use case. This post explores a pull-based master/slave architecture with master accepting work and slaves asking for work from the master. This use case is suitable for anyone who is looking to elastically provision nodes when the load is high than normal and under-provision when the load is below normal.

In this post, the master accepts RSS links from frontend which can be a user submitting links and slaves accept one RSS link and extract information such as article's published date, title and a brief description. All these information are indexed into Elasticsearch.

To begin with, the scenario looks like this:

![](/assets/scenario_1.png)

RSS links are submitted to the master and the workers register themselves with the master. Once the worker registers itself, it asks for work from the master. If the master has enough work, then it sends an RSS link to the worker which asks in a FIFO manner. 

The master/worker works according to the following protocol:

![](/assets/protocol.png)

<h4>Elastically adding nodes</h4>

Once we have the protocol established, we can add new nodes based on the load. Akka cluster membership helps here to figure which node is up. Once the node is up, the node registers itself as shown in the protocol described above. Master sends an RSS link to the worker and the worker extracts the title, description and date of each of the articles and indexes in the Elasticsearch.

Adding a node happens dynamically without altering the normal working of the cluster.
![](/assets/add_node.png)

The new node similarly works like any other node and registers itself and gets the work and starts indexing to Elasticsearch as soon as it gets some work.

<h4>Removing node(s)</h4>

![](/assets/remove_node.png)

Similar to adding node(s), removing node(s) is also dynamic. When a node is removed manually or if the node went down due to a failure, the master will get to know based on the Akka cluster membership and auto-down that worker after a while based on the akka cluster configuration. Dynamically removing node doesn't alter the normal working of the cluster. If currently a work being performed by a node which goes down due to failure, then the work assigned to that node will be marked as pending by the master since the master doesn't get an acknowledgement back by the worker. This undone work will be put back into the queue by the master and will be assigned to someother worker.

<h4>Elasticsearch indexing</h4>

Once the workers start indexing, the Elasticsearch indexing status can be viewed through marvel plugin. Marvel is an excellent plugin to view the status of the Elasticsearch's cluster health and the number of documents indexed in near-real time.

<h4>Conclusion</h4>

This use case explains a use case where adding and removing nodes is essential for dynamically changing load without altering the normal working of the cluster. Not many frameworks has such flexibility with regards to building a distributed system and Akka cluster provides a great platform to build such frameworks without the need to worry about handling the internals of building a distributed systems. This example doesn't take into account when the master fails and can be a potential bottleneck as well as a single point of failure (SPOF). This can be worked around by having multiple master-failovers which is beyond the scope of this post.

The advantage of a pull based architecture is that there is implicit throttling and each worker will asynchronously ask for more work only when it has finished its current work on hand. On the other hand, if the master pushes work to each worker, then that would lead to disastrous results unless the master implements throttling explicitly. There is no way to know how much load a worker already has if the master keeps pushing to worker. Some of the niceties available in a pull based approach would be lost in a push mechanism. Dynamically adding/removing nodes would not be as easy in a push based mechanism compared to pull based.

