---
layout: post
title:  "Akka cluster"
date:   2015-07-18 11:00:00
categories: akka, distributed systems
---

For anyone who is serious about distributed systems or building one, there are always issues that are commonly encountered.Issues such as replication, consistency, availability and partition tolerance (CAP)[1]. In a real life scenario, partition tolerance is inevitable. So the system must be able to handle partition tolerance when there are network outages. Therefore, P in the CAP is a must for any distributed system. This has been backed by Peter Deutsch in his  (<a href="https://blogs.oracle.com/jag/resource/Fallacies.html">Eight Fallacies of Distributed Computing</a>).

Now, we are left with just C and A and need to pick one of those. Some applications need strong consistency over availability. Since we are talking about Akka cluster and it is inspired from Dyanamo, it relaxes consistency and tries to be more available in case of network partitions. Akka cluster is a peer-to-peer system and uses gossip protocol for membership changes. Failure detection in the cluster is handled using 'Phi Accrual Failure Detector'. 

Gossip starts when a Member joins the cluster. It kicks off by sending Gossip messages to random nodes in the cluster while scheduling a timer. All the source code in this post is from the Akka source code [2].

{% highlight scala %}
// start periodic gossip to random nodes in cluster
val gossipTask = scheduler.schedule(PeriodicTasksInitialDelay.max(GossipInterval),
							GossipInterval, self, GossipTick)
{% endhighlight %}

The Gossip is a CRDT[3] and has a struture shown here:

{% highlight scala %}
private[cluster] final case class Gossip(
  members: immutable.SortedSet[Member], // sorted set of members with their status, sorted by address
  overview: GossipOverview = GossipOverview(),
  version: VectorClock = VectorClock()) { // vector clock version
{% endhighlight %}

Each Gossip message is sent to randomly selected nodes. The function below sends the gossip periodically which was scheduled earlier in the gossipTask.

{% highlight scala %}
def gossip(): Unit = {

	if (!isSingletonCluster) {
	  val localGossip = latestGossip

	  val preferredGossipTargets: Vector[UniqueAddress] =
	    if (ThreadLocalRandom.current.nextDouble() < adjustedGossipDifferentViewProbability) {
	      // If it's time to try to gossip to some nodes with a different view
	      // gossip to a random alive member with preference to a member with older gossip version
	      localGossip.members.collect {
	        case m if !localGossip.seenByNode(m.uniqueAddress) && validNodeForGossip(m.uniqueAddress) ⇒
	          m.uniqueAddress
	      }(breakOut)
	    } else Vector.empty

	  if (preferredGossipTargets.nonEmpty) {
	    val peer = selectRandomNode(preferredGossipTargets)
	    // send full gossip because it has different view
	    peer foreach gossipTo
	  } else {
	    // Fall back to localGossip; important to not accidentally use `map` of the SortedSet, since the original order is not preserved)
	    val peer = selectRandomNode(localGossip.members.toIndexedSeq.collect {
	      case m if validNodeForGossip(m.uniqueAddress) ⇒ m.uniqueAddress
	    })
	    peer foreach { node ⇒
	      if (localGossip.seenByNode(node)) gossipStatusTo(node)
	      else gossipTo(node)
	    }
	  }
	}
}
{% endhighlight %}

When a new member is joined, it is greeted with a Welcome message from the current Leader and the Leader sends the latest gossip seen by the entire cluster known as GossipOverview.

{% highlight scala %}
def welcome(joinWith: Address, from: UniqueAddress, gossip: Gossip): Unit = {
	require(latestGossip.members.isEmpty, "Join can only be done from empty state")
	if (joinWith != from.address)
	  logInfo("Ignoring welcome from [{}] when trying to join with [{}]", from.address, joinWith)
	else {
	  logInfo("Welcome from [{}]", from.address)
	  latestGossip = gossip seen selfUniqueAddress
	  publish(latestGossip)
	  if (from != selfUniqueAddress)
	    gossipTo(from, sender())
	  becomeInitialized()
	}
}
{% endhighlight %}

Unless the cluster converges to the same state seen by all nodes, Gossip convergence cannot happen until the node is made manually down or auto down by the Leader.

As far as the failure detection is concerned, heartbeats are sent to a few other nodes and then in turn respond back with another heartbeat.

{% highlight scala %}
// start periodic heartbeat to other nodes in cluster
val heartbeatTask = scheduler.schedule(PeriodicTasksInitialDelay max HeartbeatInterval,
	HeartbeatInterval, self, HeartbeatTick)
{% endhighlight %}


Heartbeat is sent to all the active members within the cluster:

{% highlight scala %}
def heartbeat(): Unit = {
	state.activeReceivers foreach { to ⇒
	  if (cluster.failureDetector.isMonitoring(to.address))
	    log.debug("Cluster Node [{}] - Heartbeat to [{}]", selfAddress, to.address)
	  else {
	    log.debug("Cluster Node [{}] - First Heartbeat to [{}]", selfAddress, to.address)
	    // schedule the expected first heartbeat for later, which will give the
	    // other side a chance to reply, and also trigger some resends if needed
	    scheduler.scheduleOnce(HeartbeatExpectedResponseAfter, self, ExpectedFirstHeartbeat(to))
	  }
	  heartbeatReceiver(to.address) ! selfHeartbeat
	}
}
{% endhighlight %}

And the heartbeat response is recieved from all the nodes to which it was sent:

{% highlight scala %}
private[cluster] final class ClusterHeartbeatReceiver extends Actor with ActorLogging {
  import ClusterHeartbeatSender._

  val selfHeartbeatRsp = HeartbeatRsp(Cluster(context.system).selfUniqueAddress)

  def receive = {
    case Heartbeat(from) ⇒ sender() ! selfHeartbeatRsp
  }
}
{% endhighlight %}

In the next post I will discuss some more details of the Akka cluster with some use cases along with code :)

<h4>References:</h4>

[1] CAP theorem proof from Lynch and Gilbert: <a href>http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.67.6951&rep=rep1&type=pdf</a>
[2] Akka source code: <a href>https://github.com/akka/akka/tree/master/akka-cluster/src/main/scala/akka/cluster</a>
[3] CRDT: <a href>https://hal.inria.fr/inria-00555588/document</a>
