# Designing-Data-Intensive-Applications-Summary

## Chapter 5: Replication

### Intro

Replication means keeping a copy of the same data on multiple machines.
Replica: a node that stores a copy of the database.

This chapter assumes the dataset is small such that each replica could hold the entire dataset.

Why to replicate data:
- `Decrease latency`: To keep data geographically close to your users.
- `Increase availability`: To allow the system to continue working even if some of its parts have failed.
- `Increase read throughput`: To scale out the number of machines that can serve read queries.

Problem: If data changes how to reflect this change on replicas.

#### Configuration options:
1. `Synchronous or Asynchronous Replication`
2. `Failed replica handling`

#### Replication algorithms:
1. `Single-leader`
2. `Multiple-leader`
3. `Leaderless`

---

### Leaders and Followers

Problem: Write requests need to be processed by each replica to ensure having the same data on each of them (Consistency).
Solution: Leader-based (active/passive, master/slave, primary/secondary) replication.

One node is designated to be leader (Active, Master, Primary, read-write replicas) accepts write requests updating its local storage.
Other nodes are designated to be followers (Passive, Slave, Secondaries, read-only replicas, hot standbys) recieves replication log (change stream) from leader and process writes in order in which leader processed them updating their local storage.
Client can send read requests to either leader or follower but only leader would accept write requests.

This mode of replication isn't popular only in databases but also in distributed message brokers, network filesystems and replicated block devices.

##### Side note:
`Hot Standby`: A backup server that is fully operational and runs in parallel with the primary server and takes over instantly in case of failure.  
`Warm Standby`: A backup server that is running but not actively handling requests, requiring some time to become fully operational.  
`Cold Standby`: A backup server that is powered off until needed, requiring manual intervention and a longer recovery time.

---

### Synchronous Versus Asynchronous Replication

In relational databases, this is often a configurable option while other systems have one of them hardcoded.

#### Synchronous follower
Leader sends the write to follower and waits for reponse before responding to user.
replication usually is quite fast. However, there is no guarantee of how long it might
take (replica recovering from a failure, system operating near maximum capacity or network problems).

Pros:
Follower have the up-to-date copy of data consistent with that of leader (`data reliability`).

Cons:
Write can't be processed if follower doesn't reponse forcing leader to block all other writes until follower is available again.

Thus it is not practical to have all followers synchronous (if any one fails system performance degrades substantially) instead one of them is made synchronous and if fails an asycnhronous follower is made synchronous to assure that at least two nodes (leader and synchronous follower) have up-to-date copy and hence this configuration is known as `semi-synchronous`.

#### Asynchronous follower
Leader sends the write to the follower and responds to user without waiting for confiramation from follower.

Leader-based replication usually configured as fully asynchronous.

Pros:
Leader continues processing writes even if all followers have fallen behind (as there is no synchronous follower to wait for) hence `enhanching write availability`

Cons:
If leader fails, any write that hasn't been replicated are lost `harming durability`.

Although weakening durability sounds bad however asynchronous replication is widely used, espically many followers are available. 

##### side note:
Researchers investigated methods providing good performance and availabiltiy without data loss in case of leader failure. Example, chain replication (variant of synchronous replication) used in Microsoft Azure Storage relying on consistency and consensus (nodes agreeing on a value) relation.

---

### Setting Up New Followers

Setting a new follower to increase number of replicas or replace failed nodes.
Standard file copy from node to node wouldn't work as data is always in flux (changing) and thus it only makes sense if we lock writes on leader file during copying which goes against our goal of high availability.

There is a way that we could make this feasiable without any downtime
1. Taking a snapshot at some point of time without locking entire database (feature supported by databases or extensions to handle backups).
2. Copying the snapshot to the new follower.
3. The follower connects to leader and requests changes since snapshot has been taken, Thus our snapshot needs to be associated with an exact position in leader's replication log.
4. After the follower has processed backlog since the snapshot. it has caught up with leader and can continue to process data changes from leader as they happen.

This workflow may be carried manually or automatically depending on database.

---

### Handling Node Outages

Any node can go down due to fault or system maintance. Thus our goal is to keep entire system running despite node failures and reduce impact of node outage.

Problem: Achieve high availability with leader-based replication

Solution:
#### Follower failure: Catch-up recovery
Follower keeps a log of the data changes it has received from the leader. If anything went wrong, follower can recover easily as it knows the last transaction that was processed successfully. Thus, the follower can connect to the leader and request all the data changes that occurred sfter that point. After applying these changes, it has caught up to the leader and can continue normally.

##### Leader failure: Failover
`Failover`: one of the followers needs to be promoted to be the new leader, clients need to send their writes to the new leader, and the other followers need to start listening on data changes from the new leader.

Handling leader failure is trickier than follower failure.
Failover can happen manually or automatically.

Automatic failover process:
1. `Determining leader failure`: Nodes bounce messages from time to time between each other. if node didn't respond for some time, it is declared dead.
2. `Choosing new leader`: Usually done through election process (consensus) or appointed by a previously elected controller node. The best candidate is usually replica with most up-to-date copy minimizing data loss.
3. `Reconfiguring the system`: Clients need to send their write requests to new leader, other followers need to take changes from new leader. If old leader comes back. the system must force it to step down and become a follower.

---

### Implementation of Replication Logs

`empty`

---

### Problems with Replication Lag

---

## Chapter 6: Partitioning

### Intro

`Partitioning`: each piece of data belongs exactly to one partition to support `scalability`.

Partition can be viewed as a small database though the database can support operations that affect multiple partitions.
The dataset is distributed across many disks and the query load is distributed across many processors.

Query operating on a single partition is executed independently on this partition, so throughput is scaled by adding more nodes and large complex queries can be parallelized even though this gets significantly harder.

Depending on whether the workload is OLTP or OLAB, the tunning of the system differs but the fundamentals are the same.

---

### Partitioning and Replication

Partitioning is combined by replication for `fault tolerance` as a result replication of partitions applies equally to replication of databases.

For a leader-follower replication, Each partition’s leader is assigned to one node, and its followers are assigned to other nodes (replication must be done on multiple nodes otherwise it loses its purpose). Each node may be the leader for some partitions and a follower for other partitions.

The choice of partitioning scheme is mostly independent of the choice of replication scheme.

---

### Partitioning of Key-Value Data

Problem: How to decide which records to store on which nodes?

The goal is to distribute the data and query load evenly making data handling and throughput scales linearly with the number of nodes.

The presence of an unfair (`skewed`) partition will reduce efficiency and could lead to a high load on nodes which we call `hot spots` in this case resulting in a bottleneck.

The simplest approach for avoiding hot spots is by randomly distributing data among nodes but a drawback of this approach is the need to query all nodes in parallel searching for data.
A better approach is using a key-value data model where a record is accessed by its primary key, enhancing search operation (using binary search for example).

---

### Partitioning by Key Range

Assign a continuous range of keys (min to max) for each partition. the range of keys doesn't have to be evenly spaced as data isn't evenly distributed.
Partition boundaries might be chosen manually by an administrator or automatically by a database.

---

### Partitioning by Hash of Key

To avoid skew and hot spots rather than partitioning by a range of keys, partitioning is done by a range of hashes where a hash function hashes the key (needn't be cryptographically strong) which helps distribute keys evenly.
Even if the input strings are very similar, their hashes are evenly distributed across that range of numbers (Diffusion property).

The partition boundaries can be evenly spaced, or they can be chosen pseudorandomly through `consistent hashing`.
`Consistent Hashing`: a technique to evenly distribute load without having a central control or distributed consensus through choosing random partition boundaries.

Using hash partitioning eliminates efficient range queries as data are now scattered across all the partitions (sort order is lost).
Range queries thus need to be sent to all partitions or prohibited on sorted keys used in hash partitioning.
