# Replication <!-- omit in toc -->

keeping a copy of the same data on different machines

- [why would you create replication?](#why-would-you-create-replication)
- [Leaders and followers](#leaders-and-followers)
  - [Sync vs Async Replication](#sync-vs-async-replication)
  - [Setting up new followers](#setting-up-new-followers)
  - [Handling node outages](#handling-node-outages)
    - [Follower Failure: catchup recovery](#follower-failure-catchup-recovery)
    - [Leader failure: Failover](#leader-failure-failover)

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
