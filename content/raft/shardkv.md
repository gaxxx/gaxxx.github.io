+++
title = 'ShardKV Implementation'
date = 2021-11-10
[taxonomies]
tags = ["raft", "shardkv"]
+++


[6.824 Lab 4: Sharded Key/Value Service (mit.edu)](https://pdos.csail.mit.edu/6.824/labs/lab-shard.html)

This test mainly extends kvraft with multi raft group support. It has two parts: multi raft group control and shardkv.

## Goals

- shardctl
- shardkv

### Shardctl

- The implementation is similar to kvraft, but needs to ensure proper group balancing across all shards in each Config.

### Details

- After groups join or leave, how to allocate shards across groups while minimizing data migration

### Pseudocode

```python

// Server code, handle in OnMessage section

leave_or_join_a_group()
rebalance()

```

### Detail Handling

- In the tests, we have 10 shards and 3 groups, so we can keep it simple:
    - Use consistent hashing, e.g., hash_value = 3 * group_id % 10
    - Or just round robin


### ShardKV

In multi raft groups, implement a KV store. Clients fetch the latest Config, then find the corresponding raft group to read or write data. But when a new group joins, how to balance the data.

### Details

- What if the client's Config doesn't match the group's Config?
- How does each raft group upgrade to the latest Config as a whole? How to exchange data with different groups - push or pull? Who's responsible?
- How to sync (client_id, seq_id) data

### Pseudocode

```python
// Only the leader does data updates
while is_leader:
	get_next_cfg
	client_self.prepare_upgrade // All self group members know about the upgrade
	client_others.push_data  // All data needed by other groups is pushed out
  client_self.commit_upgrade  // All self group members can upgrade now
```

### Detail Implementation

- Strongly discourage cross-group communication in OnMessage, otherwise cross-group messages fly everywhere...
- Move some client logic to the server side. In kvraft, the client's logic is: perform an operation and ensure it completes. This is perfect for Config upgrades.
- During upgrades, maintain two copies of data: old Config+data and new Config+data. This allows service during upgrades. For example, upgrading Config 8→9, some shards in a group can already serve Config-9, while others have incomplete data. Client requests should still be handled correctly. Old data can be cleaned up after upgrade.
- For (client_id, seq_id) merging, keep only the largest seq_id for each client
- For ConfigUpdate requests, configure special client_id and seq_id to ensure idempotency. For example, for 8→9 data, (client_id, seq_id) can be (group_100_to_group_101_cfg_8, 9)

## Summary

The hardest part of this test set is ensuring cross-group data synchronization. During sync, update data must also be idempotent, so bringing client logic over is very natural (though I got stuck on this step and failed on my first attempt).

[Back](/raft/)
