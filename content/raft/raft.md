+++
title = 'Raft 实现'
date = 2021-11-10
[taxonomies]
tags = ["raft"] 
+++


[6.824 Lab 2: Raft (mit.edu)](https://pdos.csail.mit.edu/6.824/labs/lab-raft.html)

[In Search of an Understandable Consensus Algorithm (mit.edu)](https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf)

[raft_diagram.pdf (mit.edu)](https://pdos.csail.mit.edu/6.824/notes/raft_diagram.pdf)

在这里，我们需要实现raft的选举， log分发， 持久化等等，实现每一步的时候，实现每一步的时候，都可能会修改之前的代码，导致之前的测试失败，所以建议是每通过一个test，都commit一次代码。

在开始实际编程之前，最好先看看要通过的test实现，在test_test.go里面，其中主要的检测函数有

- checkOneLeader 在所有的peers中，有且只有一个leader
- checkTerm 返回当前的term
- one 确认该command在日志中达成共识，在这之前command，会先进行分发，然后commit，最后apply。

同时我们需有确定peer之间怎么建立通信

1. 通过一个长链接，这样比较容易维护状态，可以保证在同一个链接中，message的顺序，但是问题在于如果有一个message处理满了，会影响其他的message
2. 通过短链， 这样写起来简单，但是需要处理消息顺序的问题，我们需要在消息中加上一些序号。碰到网络断开情况的时候，可能会建立特别多的短链接。
3. 通过连接池，也需要处理消息顺序，但是可以对不同的消息配置不同的并发度，但是复杂性较高。

## 目标

- 选举 (2A)
- 数据分发 （2B）
- 持久化 （2C）
- Snapshot (2D)

## 选举 (Election)

 Election是peers选主的过程，当peer选主完成并变成leader之后，就由它向其他peer分发log数据。

Raft的选举，其实跟现实挺像的，可以与干部选举对应起来

- 3个peer准备election ——- 3个同学开始竞选班干部
- 最大的term才是有效的 ———- 我们要的是最新一届的结果
- 获得大多数同意即可 ———- 需要拿到多数票
- 同意的条件是candidate peer的log数量多 ———- 同意的条件是，参选人的经历（log）比你全.

### 细节

- 在Vote过程中term的处理
- Peer怎么投票

### 伪代码

```python
// 对于每一个Raft对象，我们需要定时检查状态， 并进行选举。

while not_killed:

	if state == follower and recv_no_heart_beat > ELECTION_TIMEOUT:

		send_vote_request_to_all
```

```python
// 同时我们需要处理其他peer的vote

if has_more_logs:

	return vote_granted

return vote_fail
```

```python
// 以及拿到的vote reply之后

save_vote_reply_for_this_peer

check_if_being_leader_now
```

### 细节处理 {#raft-detail}

1. 在发送和接受vote的时候，理想情况下在同一个term。但是实际上并不会，对于低term的忽略掉或者拒绝，对于高term的VoteRequest，可以先把自己的term调上来，清空自己当前的拿到的votes，然后处理。
2. 每个peer在本term不能同时投两个人。 （可以弃权，然后换一个，弃权的条件见3a)
3. 对于同一个term，每个peer的投票策略有挺多的，跟现实也挺像的
    1. 参选的人，先投自己，然后去别人那边拉票。这里还有一个增强，在投票过程中，发现自己拿到的票，已经证明自己没法当选了，可以提前弃权，并开始投别人的票 
    2. 先拉票，在最后阶段，投自己一票 (在我的机器上，这个策略要快10%左右）
4. 在candiate状态时，可以配置一个定时器来进行election， 如果正常的leader碰到了一个乱入的VoteRequest, 可以在回退到follower，并在下一个心跳，开始election。 对于其他的follower，在收到乱入的VoteRequest的时候，可以选择不处理，参考第2条。
    
    

## 数据分发

当peer变成leader之后，就可以在当前的term，分发数据了。

### 细节 

- 什么时候存数据，拿到数据就存，还是批处理？  
- 如何快速的update_index，如何快速协商几次后，给其他peer最新的数据
- Raft的文档里面说了，只commit当前term的数据，但是如果最后一个数据是之前term的，并且没有新的数据过来。 那这个数据能不能commit
- 数据commit之后，怎么apply

### 伪代码

```bash
// 开始接收数据

while not_killed:

	recv_a_log

	send_to_peers
```

```bash
// 接收peer的反馈

update_log_indexes

update_commit_info
```

```python
// 在这个阶段，所有peer都要对数据进行apply了

while not_killed:

	get_last_applied_msg
	
	do_apply
```

### 细节处理 {#when-to-save}

- 拿到数据就存，否则一次两次可能可以过，但是跑10次就会跪那么一两次的。但是存的时候，需要看是存raft logs,还是连着snapshot一起。需要注意的时候，因为存的时候需要序列化，所以后面kvraft的test在性能比较低的机器上，比如我的树莓派上，有些会跪掉。
- 如何快速的update index, 得看返回AppendEntriesReply是否有conflict
    - 如果没有conflict
        - 更新该peer的最后一个item （log_index + term)
        - 搜集所有peer的最后一个item，如果是当前term，通过判断大多数达到的位置，update commit pos
        - 如果所有peer的最后一个item，都是一个旧的term，则取所有peer的index最小值， update commit pos
    - 如果有conflict （conflict_log_idx, conflict_log_term)
        - 如果conflict_log_term与当前conflict_log_idx的一致，可以配置next_match = conflict_log_idx  + 1
        - 如果不一致，next_match = conflict_log_idx, 或者可以配置成 next_match = conflict_log_idx - N (N 可以自己配置，比如当前的log数量的一半，或者是其他条件），也可以按照term的信息来配置，比如
- 数据commit之后，我们可以来进行apply，我最开始的做法是
    
    ```python
    for last_applied < commited:
    
    	do_apply(logs[last_applied])
    
    	last_applied++
    ```
    
    但是这样可能会有锁的问题，另外如果log里面有snapmsg， 可能log会发生截断，但是对于log本身来说，是不可变的，所以我个人倾向于下面这种，而且锁的粒度很好控制
    
    ```go
    cmd, ok := get_to_apply(last_applied)
    
    do_apply(cmd)
    
    last_appied = cmd.idx
    ```
    

## 持久化

在测试和实际产品环境中， raft的数据随时都可以失效，我们需要保证的是在任何时候， Snapshot + LogApply最终达到一致，所以需要存储的时候，包括LogState + SnapState. 但是在这个测试里面，没有Snap。我们只需要存所有的Log就可以了。

### 细节

- 存那些State，Raft的文档里面只有这些，还需要其他吗？

![Untitled](/raft/raft_persist.png)

### 伪代码

```python
// when killed
saved_data = rf.Data.Encode() // term, vote, log

// when restore
rf.Data.Decode(saved_data)
```

### 细节实现

- 存储的数据，如果Log不是从0开始的，还需要 EntryIdx, EntryTerm, SnapIdx, SnapTerm.

## Snapshot

Snapshot保证了，在Snapshot + (SnapIdx ... CommitIdx).apply() 的结果，和 （0 .. CommitIdx).apply() 的是一致的

### 细节

- Snapshot什么时候生成
- Snapshot什么时候发送
- 收到了Snapshot怎么办

### 伪代码

```python
// 收到Snap请求,因为snap_data需要外部状态机提供
persist(snap_data, snap_idx）
clear_useless_logs()

// 发送heartbeat的是否，如果发现对方peer要的log，本地已经没有了
if log_not_exists:
	send_snap_data()

// 关闭服务的时候
persist(snap_data, raft_data)
```

### 细节处理

- Raft可以把自己的状态暴露出去，比如当前log的size， 这样外部随时都可以来做Snapshot。 做Snapshot的时候，外部会把它已经applied的idx和最新状态传进来。这样Raft服务需要存储的数据有：
    - SnapState, SnapIdx, SnapTerm (optional)
    - Logs not applied
    
    存储的时候，不一定要把所有的applied log处理掉，可以保留一部分。
    
- 接上一条，如果把所有applied log处理掉了，那这些日志没有分发到其他的peer（少于1/2）的时候， 那我们就只能发送Snapshot了。
- 收到了Snapshot之后， 最好是先存储， 跟commit的逻辑是一样的。
    
    

## 总结 {#avoid-lock}

这个项目是其他项目的基础， 其中有很多tech decision是在做后面的项目之后，慢慢总结出来的。 

在与外部系统交互时， 这个模块与外面的通信应该保持最小的接口. 

- Start / Kill
- Snapshot
- OnMessage

如果外部信息希望更多的了解Raft的数据，可以通过传入控制型的command，比如说 RaftState{},

然后Raft在把这个信息传给状态机之前，可以填充一些内部信息。最后得到Raft的内部状态

RaftState {

     LogInfo:

     SnapInfo:

     TotalIdx: 

}

同时，Raft也可以主动对外通信，发送一些状态变化，比如 BecomingALeader等等。。。

通过这些方式，就可以避免各种死锁问题。

[Back](/raft/)
