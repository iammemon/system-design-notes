# Replication <!-- omit in toc -->

keeping a copy of the same data on different machines

- [why would you create replication?](#why-would-you-create-replication)
- [Leaders and followers](#leaders-and-followers)
  - [Sync vs Async Replication](#sync-vs-async-replication)
  - [Setting up new followers](#setting-up-new-followers)
  - [Handling node outages](#handling-node-outages)
    - [Follower Failure: catchup recovery](#follower-failure-catchup-recovery)
    - [Leader failure: Failover](#leader-failure-failover)
  - [Implementation of replication logs](#implementation-of-replication-logs)
    - [Statement-based replication](#statement-based-replication)
    - [Write-ahead log (WAL) shipping](#write-ahead-log-wal-shipping)
    - [Logical (row-based) log replication](#logical-row-based-log-replication)
    - [Trigger-based replication](#trigger-based-replication)
  - [Problems with replication lag](#problems-with-replication-lag)
    - [Reading your own writes](#reading-your-own-writes)
    - [Monotonic Reads](#monotonic-reads)
    - [Consistent Prefix Reads](#consistent-prefix-reads)
    - [Solution for replication Lag](#solution-for-replication-lag)

## why would you create replication?

- geographical distribution of data closer to your users (reduce latency)
- avoid single point of failure when a node goes down other can be used to serve data (increase availability )
- Multiple machines to serve read queries (Increase read throughput)

## Leaders and followers

### Sync vs Async Replication

- **Sync**: follower is gauranteed to have up-to-date copy that is consistent with leader. leader wait's for follower confirmation and then report write success.

  > Note: not practical because if a follower is down it can halt's the transaction and leader will block all writes

- **Semi-sync**: one follower is sync and other is async. if the sync follower is slow or down other async follower is made sync.guarantess up-to-date copy on at least 2 nodes (leader,sync follower)
- **Async**: leader continues the writes without follower nodes confirmation even if all the follower nodes are down. drawback is if leader goes down the writes that are not replicated are lost (Weak durability) but overall this configuration is widely used

### Setting up new followers

simply copying data files from one node to another is typically not sufficient because writing can happen in the background while the replication is in progress and we don't want to lock the DB because that will compromise high availability

The following is the conceptual process:

- Take a consistent snapshot of the leader database without locking
- Copy the snapshot to the follower node
- Follower requests data changes that have happened since the snapshot was taken
- Once follower processed the backlog of data changes since snapshot, it has _caught up_.

### Handling node outages

how do you achieve high availability with leader based replication

#### Follower Failure: catchup recovery

Follower can connect to the leader and request all the data changes that occurred during the time when the follower was disconnected.It maintains a log that allow the follower to know about it's last transaction

#### Leader failure: Failover

One of the followers needs to be promoted to be the new leader, clients need to be reconfigured to send their writes to the new leader and followers need to start consuming data changes from the new leader.

Automatic failover process:

- Determining that the leader has failed. If a node does not respond in a period of time it's considered dead.Most system use a timeout and nodes frequently bounce message back and forth b/w each other
- Choosing a new leader. The best candidate for leadership is usually the replica with the most up-to-date changes from the old leader.
- Reconfiguring the system to use the new leader. Clients now need to send their write request to the new leader (Request Routing) The system needs to ensure that the old leader when comes back becomes a follower and recognises the new leader

Things that could go wrong:

- If asynchronous replication is used, the new leader may have received conflicting writes in the meantime. The common solution here is to discard old leader unreplicated writes which may voilate client's durability expectation
- Discarding writes is especially dangerous if other storage systems outside of the database need to be coordinated with the database contents. for example cache server/DB might have the records that was discarded that will create inconsistency
- It could happen that two nodes both believe that they are the leader (split brain). Data is likely to be lost or corrupted.
- What is the right timeout before the leader is declared dead?
  - too long means longer time to recover
  - too short could cause unnecessary failovers in temp load spike or network glitch

For these reasons, some operation teams prefer to perform failovers manually, even if the software supports automatic failover.

### Implementation of replication logs

Understanding different methods that are used for leader based replication

#### Statement-based replication

logs wirte requests/statement (INSERT,UPDATE,DELETE) and send it to followers so they parse and executes those statement

Issues with this approach:

- Non deterministic function will generate different value on each node such as `Now()` or `RAND()`
- Statements that depend on existing data, like auto-increments, must be executed in the same order in each replica.
- Statement with side effects like triggers,stored procedures or user-defined functions may produce different result on each replica

A possible solution is to replace any nondeterministic function with a fixed value

> Note: Because there are so many edge cases with this other methods are generally prefered

#### Write-ahead log (WAL) shipping

The log is append-only sequence of bytes that contain all the writes to the database.The leader send this to all followers

The main disadvantage is that the log describes the data at a very low level (like which bytes were changed in which disk blocks), coupling it to the storage engine.

Usually is not possible to run different versions of the database in leaders and followers. This can have a big operational impact, like making it impossible to have a zero-downtime upgrade of the database.

> Note: Oracle and PostgreSQL use this method for replication

#### Logical (row-based) log replication

Basically a sequence of records describing writes to database tables at the granularity of a row

- For an inserted row, the new values of all columns.
- For a deleted row, the information that uniquely identifies that column.
- For an updated row, the information to uniquely identify that row and all the new values of the columns.

A transaction that modifies several rows, generates several of such logs, followed by a record indicating that the transaction was committed. MySQL binlog uses this approach.

Since logical log is decoupled from the storage engine internals, it's easier to make it backwards compatible.

Logical logs are also easier for external applications to parse, useful for data warehouses, custom indexes and caches (change data capture).

> Note: MYSQL binlog (when configure) uses this approach

#### Trigger-based replication

There are some situations were you may need to move replication up to the application layer.

A trigger lets you register custom application code that is automatically executed when a data change occurs. This is a good opportunity to log this change into a separate table, from which it can be read by an external process.

Main disadvantages is that this approach has greater overheads, is more prone to bugs but it may be useful due to its flexibility.

### Problems with replication lag

Node failures is just one reason for wanting replication. Other reasons are scalability and latency.

In a read-scaling architecture, you can increase the capacity for serving read-only requests simply by adding more followers. However, this only realistically works on asynchronous replication. The more nodes you have, the likelier is that one will be down, so a fully synchronous configuration would be unreliable.

With an asynchronous approach, a follower may fall behind, leading to inconsistencies in the database (eventual consistency).

The problems that may arise and how to solve them.

#### Reading your own writes

Read-after-write consistency, also known as read-your-writes consistency is a guarantee that if the user reloads the page, they will always see any updates they submitted themselves.

Possible techiniques to implement read-after-write consistency

- When reading something that the user may have modified, read it from the leader. For example, user profile information on a social network is normally only editable by the owner. A simple rule is always read the user's own profile from the leader.
- You could track the time of the latest update and, for one minute after the last update, make all reads from the leader
- The client can remember the timestamp of the most recent write, then the system can ensure that the replica serving any reads for that user reflects updates at least until that timestamp. The timestamp can be logical timestamp (log sequence number)
- If your replicas are distributed across multiple datacenters, then any request needs to be routed to the datacenter that contains the leader.

Another complication is that the same user is accessing your service from multiple devices, you may want to provide cross-device read-after-write consistency.

Some additional issues to consider:

- Remembering the timestamp of the user's last update becomes more difficult. The metadata will need to be centralised.
- If replicas are distributed across datacenters, there is no guarantee that connections from different devices will be routed to the same datacenter. You may need to route requests from all of a user's devices to the same datacenter.

#### Monotonic Reads

Because of followers falling behind, it's possible for a user to see things moving backward in time.

When you read data, you may see an old value; monotonic reads only means that if one user makes several reads in sequence, they will not see time go backward.

Make sure that each user always makes their reads from the same replica. The replica can be chosen based on a hash of the user ID. If the replica fails, the user's queries will need to be rerouted to another replica.

#### Consistent Prefix Reads

If a sequence of writes happens in a certain order, then anyone reading those writes will see them appear in the same order.

This is a particular problem in partitioned (sharded) databases as there is no global ordering of writes.

A solution is to make sure any writes casually related to each other are written to the same partition.

#### Solution for replication Lag

Transactions exist so there is a way for a database to provide stronger guarantees so that the application can be simpler.
