+++
title = 'Kvraft 实现'
date = 2021-11-10
[taxonomies]
tags = ["kvraft", "raft"] 
+++

[6.824 Lab 3: Paxos-based Key/Value Service (mit.edu)](http://nil.csail.mit.edu/6.824/2015/labs/lab-3.html)

[Bolosky.pdf (usenix.org)](https://static.usenix.org/event/nsdi11/tech/full_papers/Bolosky.pdf)

在kvraft这个系统里面，主要是处理 GET / PUT / APPEND的请求，要通过的测试，对于这些请求有如下的约束

- 客户端是一个一个发送请求的
- 对于客户端的 PUT / APPEND 可能会出现网络错误，无法到达服务端，也可能多次到达服务端，服务器应该能保证exactly once.

## 目标

- 完成所有的Test

## 完成所有的Test

- 第一步，完成TestBasic3A， 这个测试里面，每个client写不同的key，然后多次迭代，看看每次写入和读取的是否一致。
- 在Snap的时候，把所有的状态就写进去就可以了。

### 细节

- 对于server端来说，如何存储key / value。
- 对于Get请求，需要把它放到log里面处理吗
- 如果把所有的请求都通过Start放入到Log里面， 什么时候apply，apply之后怎么返回。

### 伪代码

```python
// 客户端代码

while not_complete:
	select_peer
	rpc_call_cmd()
  if errcode == ErrWrongLeader / ErrTimeout:
		continue
  return result

// 服务器代码
cmd = get_a_cmd()
start(cmd)
......
wait cmd
......
	if is_timeout:
		return ErrTimeout
return cmd.result

    
```

### 细节处理

- 所有的请求，都需要放到logs里面，然后在apply的时候处理. Apply的时候，因为是在单独的线程，如果其实 key / value 直接用map就行。
- 对于Put 和 Append请求，我们需要为每个client配置一个incremental seq id, 然后在服务端处理的时候保证，对于同一个client id, seq id 是递增的，在这个case里面，应该是每次都是喜加一。
- 对于已经执行过的PUT / APPEND 请求 (seq id比当前的小）， 可以直接返回成功。
- 在服务器端维护一个 (log_idx, log_term) → (chan result) 的map， apply处理结束之后，给chan result发送消息，然后结束该rpc
- 可以把 (client_id , seq_id) 的逻辑封装在内部，对外只暴露 OnMessage / OnSnapshot 的接口。

## 总结

kvraft里面主要增加了状态处理完成之后，给客户端应答的工作。如果客户端是一个长链接，可以异步的返回处理结果，这个就跟zk比较类似了。

[Back](/raft/)
