+++
title = 'ShardKv 实现'
date = 2021-11-10
[taxonomies]
tags = ["raft", "shardkv"] 
+++


[6.824 Lab 4: Sharded Key/Value Service (mit.edu)](https://pdos.csail.mit.edu/6.824/labs/lab-shard.html)

这个测试，主要是在kvraft的基础下，扩展了multi raft group. 它分成两个部分，一个是 mutli raft group control, 另外一个部分是 shardkv.

## 目标

- shardctl
- shardkv

### Shardctl

- 这个的实现与kvraft类似，但是需要保证每次Config里面，所有shard之间的group如何balance。

### 细节

- group加入，或者离开之后， 所有Shard的group怎么分配，保证数据的迁移量最小

### 伪代码

```python

// 服务器代码， 在OnMessage部分进行处理

leave_or_join_a_group()
rebalance()
    
```

### 细节处理

- 在测试里面，我们有10个shard，3个group，所以可以搞简单一点
    - 采用consistency hash, 例如 hash_value = 3 * group_id % 10
    - 或者直接round robin
    

### ShardKv

在multi raft group中，实现一个kv store的功能， 客户端会取最新的Config，然后找到对应的 raft group获取或者写入数据。但是如果这个时候，新加入一个group，如何来进行数据的balance。

### 细节

- 客户端的Config与它访问的group拿到的Config不一致怎么处理？
- 每个raft group怎么整体升级到最新的Config，如何与不同的group交换数据，是把数据推出来，还是拉过来？谁来负责这个事情
- (client_id, seq_id) 的数据怎么同步

### 伪代码

```python
// 只有leader才会去做数据的update
while is_leader:
	get_next_cfg
	client_self.prepare_upgrade // 所有的self group都知道要升级了
	client_others.push_data  // 所有的其他的group需要的数据都推出去了
  client_self.commit_upgrade  // 所有的self group都可以升级了
```

### 细节实现

- 非常不建议在OnMessage里面去做跨group之间的通信，否则的话，跨group的message满天飞。。。
- 把Client的部分逻辑，拿到Server端，kvraft里面， client的实现是，做一个操作，并保证它完成，这个非常适合升级Config。
- 在升级的过程中，我们可以保存两份数据， 旧的Config和data，新的Config和data。 这样在升级的过程中， 也可以提供服务。 比如 Config 升级 8→9,  这个时候有个Group的部分shard已经可以提供 Config-9的服务了，但是其他shard的数据不全。这个时候， client的请求也应该能正确的处理， 旧数据可以在升级后清理掉。
- (client_id, seq_id) 的合并，对于每个client，可以只保留最大的 seq_id
- 在ConfigUpdate的这些请求，我们可以为他们配置特殊的client_id 和 seq_id,来保证幂等，比如对于8-9的数据，（client_id, seq_id) 可以写成 (group_100_to_group_101_cfg_8, 9)

## 总结

这组测试最难的就是如何保证跨group的数据进行同步。 而且在同步这些数据的过程中，我们需要保证 update数据也是幂等的，所以把client的逻辑拿过来是一件非常自然的事情 （虽然我第一次卡在这一步直接跪了）

[Back](/raft/)
