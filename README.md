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
If leader fails, any write that hasn't been replicated are lost (`harming durability`).
