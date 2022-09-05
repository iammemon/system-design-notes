# Replication <!-- omit in toc -->

keeping a copy of the same data on different machines

- [why would you create replication?](#why-would-you-create-replication)
- [Leaders and followers](#leaders-and-followers)
  - [Sync vs Async Replication](#sync-vs-async-replication)

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
