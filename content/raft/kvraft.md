+++
title = 'KVRaft Implementation'
date = 2021-11-10
[taxonomies]
tags = ["kvraft", "raft"]
+++

[6.824 Lab 3: Paxos-based Key/Value Service (mit.edu)](http://nil.csail.mit.edu/6.824/2015/labs/lab-3.html)

[Bolosky.pdf (usenix.org)](https://static.usenix.org/event/nsdi11/tech/full_papers/Bolosky.pdf)

In the kvraft system, we mainly handle GET / PUT / APPEND requests. The tests have the following constraints:

- Clients send requests one at a time
- PUT / APPEND requests may fail due to network errors (not reaching the server) or arrive multiple times; the server should guarantee exactly-once semantics

## Goals

- Pass all tests

## Passing All Tests

- First step: pass TestBasic3A. This test has each client writing different keys across multiple iterations, checking if reads and writes are consistent.
- For snapshots, just write all state into them.

### Details

- How to store key/value on the server side
- Does a Get request need to go through the log?
- If all requests go through Start into the log, when to apply, and how to return results after applying

### Pseudocode

```python
// Client code

while not_complete:
	select_peer
	rpc_call_cmd()
  if errcode == ErrWrongLeader / ErrTimeout:
		continue
  return result

// Server code
cmd = get_a_cmd()
start(cmd)
......
wait cmd
......
	if is_timeout:
		return ErrTimeout
return cmd.result


```

### Detail Handling

- All requests need to go through logs, then get processed during apply. Since apply runs in a separate thread, key/value storage can simply use a map.
- For Put and Append requests, configure an incremental seq id for each client. On the server side, ensure that for the same client id, seq id is monotonically increasing - in this case, incrementing by one each time.
- For already-executed PUT / APPEND requests (seq id smaller than current), return success directly.
- Maintain a (log_idx, log_term) → (chan result) map on the server. After apply processing, send a message to chan result to end the RPC.
- You can encapsulate the (client_id, seq_id) logic internally, only exposing OnMessage / OnSnapshot interfaces externally.

## Summary

KVRaft mainly adds the work of responding to clients after state processing. If the client uses a long connection, results can be returned asynchronously, similar to ZooKeeper.

[Back](/raft/)
