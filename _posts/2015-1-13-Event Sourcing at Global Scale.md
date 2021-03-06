---
layout: post
title: EVENT SOURCING AT GLOBAL SCALE
author: mkrasser
tag: eventuate
---

We recently started to explore several options how to globally distribute an application that is based on <a href="http://martinfowler.com/eaaDev/EventSourcing.html">event sourcing</a>. The main driver behind this initiative is the requirement that geographically distinct locations (called sites) shall have low-latency access to the application: each site shall run the application in a near data center and application data shall be replicated across all sites. A site shall also remain available for writes if there are inter-site network partitions. When a partition heals, updates from different sites must be merged and conflicts (if any) resolved.

In this blog post, I’ll briefly summarize our approach. We also validated our approach with a prototype that we <a href="https://github.com/RBMHTechnology/eventuate">recently open-sourced</a>. During this year, we’ll develop this prototype into a production-ready toolkit for event sourcing at global scale.

As a starting point for our prototype, we used <a href="http://doc.akka.io/docs/akka/2.3.8/scala/persistence.html">akka-persistence</a> but soon found that the conceptual and technical extensions we needed were quite hard to implement on top of the current version of akka-persistence (2.3.8). We therefore decided for a lightweight re-implementation of the akka-persistence API (with some modifications) together with our extensions for geo-replication. Of course, we are happy to contribute our work back to akka-persistence later, should there be broader interest in our approach.

The extensions we wrote are not only useful in context of geo-replication but can also be used to overcome some of the current limitations in akka-persistence. For example, in akka-persistence, event-sourced actors must be cluster-wide singletons. With our approach, we allow several instances of an event-sourced actor to be updated concurrently on multiple nodes and conflicts (if any) to be detected and resolved. Also, we support event aggregation from several (even globally distributed) producers in a scalable way together with a deterministic replay of these events.

The following sections give an overview of the system model we developed. It is assumed that the reader is already familiar with the basics of <a href="http://martinfowler.com/eaaDev/EventSourcing.html">event sourcing</a>, <a href="http://martinfowler.com/bliki/CQRS.html">CQRS</a> and <a href="http://doc.akka.io/docs/akka/2.3.8/scala/persistence.html">akka-persistence</a>.

<h2>SITES</h2>
In our model, a geo-replicated event-sourced application is distributed across sites where each site is operated by a separate data center. For low-latency access, users interact with a site that is geographically close.

Application events generated on one site are asynchronously replicated to other sites so that application state can be reconstructed on all sites from a site-local event log. A site remains available for writes even if it is partitioned from other sites.

<h2>EVENT LOG</h2>
At the system’s core is a globally replicated event log that preserves the happened-before relationship (= potential causal relationship) of events. Happened-before relationships are tracked with vector timestamps. They are generated by one or more <a href="http://en.wikipedia.org/wiki/Vector_clock">vector clocks</a> on each site and stored together with events in the event log. By comparing vector timestamps, one can determine whether any two events have a happened-before relationship or are concurrent.

The partial ordering of events, given by their vector timestamps, is preserved in each site-local copy of the replicated event log: if <code>e1 -&gt; e2</code> then <code>offset(e1) &lt; offset(e2)</code>, where <code>-&gt;</code> is the happened-before relationship and <code>offset(e)</code> is the position or index of event <code>e</code> in a site-local event log. For example, if site A writes event <code>e1</code> that (when replicated) causes an event <code>e2</code> on site B, then the replication protocol ensures that <code>e1</code> is always stored before <code>e2</code> in <em>all</em> site-local event logs.  As a direct consequence of that storage order, applications that produce to and consume from the event log experience event replication as reliable, causally ordered event multicast: if <code>emit(e1) -&gt; emit(e2)</code> then all applications on all sites will consume <code>e1</code> before <code>e2</code>, where <code>-&gt;</code> is the happens-before relationship and <code>emit(e)</code> writes event <code>e</code> to the site-local event log.

The relative position of concurrent events in a site-local event log is not defined i.e. concurrent events may have a different ordering in different site-local event logs. Their replay, however, is deterministic per site, as a site-local event log imposes a total ordering on local event copies (which is helpful for debugging purposes, for example). A global total ordering of events is not an option in our case, as it would require global coordination which is in conflict with the availability requirement of partitioned sites. It would furthermore increase write latencies significantly.

In our implementation, we completely separate inter-site event replication from (optional) intra-site event replication. We use intra-site replication only for stronger durability guarantees i.e. for making a site-local event log highly available. We implemented asynchronous inter-site replication independent from concrete event storage backends such as <a href="https://github.com/google/leveldb">LevelDB</a>, <a href="http://kafka.apache.org/">Kafka</a>, <a href="http://cassandra.apache.org/">Cassandra</a> or whatever. This allows us to replace storage backends more easily whenever needed.

<h2>EVENT-SOURCED ACTORS</h2>

We distinguish two types of actors that interact with the event log: <a href="http://rbmhtechnology.github.io/eventuate/architecture.html#event-sourced-actors"><code>EventsourcedActor</code></a> and <a href="http://rbmhtechnology.github.io/eventuate/architecture.html#event-sourced-views"><code>EventsourcedView</code></a>. They correspond to <code>PersistentActor</code> and <code>PersistentView</code> in akka-persistence, respectively, but with a major difference in event consumption.

Like in akka-persistence, <code>EventsourcedActor</code>s (EAs) produce events to the event log (during command processing) and consume events from the event log. A major difference is that EAs do not only consume events they produce themselves but also consume events that other EAs produce to the same event log (which can be customized by filter criteria). In other words, EAs do not only consume events to reconstruct internal state but also to collaborate with each other by exchanging events which is at the heart of <a href="http://en.wikipedia.org/wiki/Event-driven_architecture">event-driven architectures</a> and <a href="http://martinfowler.com/eaaDev/EventCollaboration.html">event collaboration</a>. 

From this perspective, a replicated event log is the backbone of a distributed, durable and causality-preserving event bus that also provides the full history of events, so that event consumers can reconstruct application state any time by replaying events. For exchanging events, EAs may be co-located at the same site (Fig. 1) or distributed across sites (Fig. 2)

![Intra-site EA collaboration]({{ site.baseurl }}/images/intra-site.png)<br>
Fig. 1: Intra-site EA collaboration

![Inter-site EA collaboration]({{ site.baseurl }}/images/inter-site.png)<br>
Fig. 2: Inter-site EA collaboration


We think that our distributed event bus might be an interesting implementation option of Akka’s <a href="http://doc.akka.io/docs/akka/2.3.8/scala/event-bus.html">event bus</a>, especially for distributed event-based collaboration in an Akka <a href="http://doc.akka.io/docs/akka/2.3.8/scala/cluster-usage.html">cluster</a>. In this case, Akka cluster applications could also rely on causal ordering of events. 

One special mode of collaboration is state replication: EA instances of the same type consume each other’s events to reconstruct application state on different sites (more on that later). A related example is to maintain hot-standby instances of EAs on the same site to achieve fail-over times of milliseconds. Another example of collaboration is a distributed business process: EAs of different type process each other’s events to achieve a common goal. Reliability of the distributed business process is given by durability of events in the event log and event replay in case of failures. 

For sending messages to other non-event-sourced actors (external services, …), EAs have several options:

<ul>
  <li>during command processing with at-most-once message delivery semantics. The same option exists for <code>PersistentActor</code>s in akka-persistence by using a custom <a href="http://doc.akka.io/docs/akka/2.3.8/scala/persistence.html#event-sourcing"><code>persist</code></a> handler.</li>
  <li>during event processing with at-least-once message delivery semantics. The same option exists for <code>PersistentActor</code> in akka-persistence by using the <a href="http://doc.akka.io/docs/akka/2.3.8/scala/persistence.html#at-least-once-delivery"><code>AtLeastOnceDelivery</code></a> trait.</li>
  <li>during event processing with at-most-once message delivery semantics. The same option exists for <code>PersistentActor</code> in akka-persistence by checking whether a consumed event is a live event or a replayed event. </li>
</ul>

Replies from external services are processed like external commands: they may produce new events which may trigger the next step in a distributed business process, for example.

Since EAs can consume events from other EAs, they can also generate any view of application state. An EA can consume events from all other globally or locally distributed producers (EAs) by consuming from the shared, replicated event log. This overcomes a current limitation in akka-persistence where events cannot be easily aggregated from several producers (at least not in a scalable and deterministic way).

If an application wants to restrict an actor to only consume from the event log it should implement the <code>EventsourcedView</code> (EV) trait (instead of <code>EventsourcedActor</code>) which implements only the event consumer part of an EA. From a CQRS perspective, 

<ul>
  <li>EAs should be used to implement the command side (C) of CQRS and maintain a write model (in-memory only in our application)</li>
  <li>EVs should be used to implement the query side (Q) of CQRS and maintain a read model (in-memory or persistent in our application)</li>
</ul>

In addition to EAs and EVs, we also plan to implement an interface to <a href="http://doc.akka.io/docs/akka-stream-and-http-experimental/1.0-M2/scala.html">akka-streams</a> for producing to and consuming from the distributed event log.

<h2>STATE REPLICATION</h2>

As already mentioned, by using a replicated event log, application state can be reconstructed (= replicated) on different sites. During inter-site network partitions, sites must remain available for updates to replicated state. Consequently, conflicting updates may occur which must be detected and resolved later when the partition heals. More precisely, conflicting updates may also occur without an inter-site network partition if updates are concurrent.

An example: site A makes an update to the replicated domain object <code>x1</code> and the corresponding update event <code>e1</code> is written to the replicated event log. Some times later, site A receives another update event <code>e2</code> for the same domain object <code>x1</code> from site B. If site B has processed event <code>e1</code> before emitting <code>e2</code>, then <code>e2</code> causally depends on <code>e1</code> and site A can simply apply <code>e2</code> to update <code>x1</code>. In this case, the two updates, represented by <code>e1</code> and <code>e2</code>, have been applied to the replicated domain object <code>x1</code> on both sites and both copies of <code>x1</code> converge to the same value. On the other hand, if site B concurrently made an update to <code>x1</code> (be it because of a network partition or not), there might be a conflict.

Whether concurrent events are also conflicting events completely depends on application logic. For example, concurrent updates to different domain objects may be acceptable to an application whereas concurrent updates to the same domain object may be considered as conflict and must be resolved. Whether any two events are concurrent or have a happened-before relationship (= a potential causal relationship) can be determined by comparing their vector timestamps.

<h2>CONFLICT RESOLUTION</h2>

If application state can be modeled with <a href="http://rbmhtechnology.github.io/eventuate/user-guide.html#commutative-replicated-data-types">commutative replicated data types</a> (CmRDTs) alone, where state update operations are replicated via events, concurrent updates are not an issue at all. However, many state update operations in our application do not commute and we support both interactive and automated conflict resolution strategies.

Conflicting versions of application state are tracked in a concurrent versions tree (where the tree structure is determined by the vector timestamps of contributing events). For any state value of type <code>S</code> and updates of type <code>A</code>, concurrent versions of <code>S</code> can be tracked in a generic way with data type <a href="http://rbmhtechnology.github.io/eventuate/user-guide.html#tracking-conflicting-versions"><code>ConcurrentVersions</code></a>. Concurrent versions can be tracked for different parts of application state independently, such as individual domain objects or even domain object fields, depending on which granularity level an application wants to detect and resolve conflicts. 

During <a href="http://rbmhtechnology.github.io/eventuate/user-guide.html#interactive-conflict-resolution">interactive conflict resolution</a>, a user selects one of the conflicting versions as the “winner”. This selection is stored as explicit conflict resolution event in the event log so that no further user interaction is needed during later event replays. A possible extension could be an interactive merge of conflicting versions. In this case, the conflict resolution event must contain the merge details so that the merge is reproducible.

<a href="http://rbmhtechnology.github.io/eventuate/user-guide.html#automated-conflict-resolution">Automated conflict resolution</a> applies a custom conflict resolution function to conflicting versions in order to select a winner. A conflict resolution function could also automatically merge conflicting versions but then we are already in the field of <a href="http://en.wikipedia.org/wiki/Conflict-free_replicated_data_type#State-based_CRDTs">convergent replicated data types</a> (CvRDTs). 

