+++
title = 'Raft Implementation'
date = 2021-11-10
[taxonomies]
tags = ["raft"]
+++


[6.824 Lab 2: Raft (mit.edu)](https://pdos.csail.mit.edu/6.824/labs/lab-raft.html)

[In Search of an Understandable Consensus Algorithm (mit.edu)](https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf)

[raft_diagram.pdf (mit.edu)](https://pdos.csail.mit.edu/6.824/notes/raft_diagram.pdf)

Here we implement Raft's election, log distribution, persistence, etc. Each step may require modifying previous code, causing earlier tests to fail. It's recommended to commit code after passing each test.

Before starting to code, review the tests in test_test.go. The main check functions are:

- checkOneLeader - among all peers, there is exactly one leader
- checkTerm - returns the current term
- one - confirms that a command reaches consensus in the log. Before this, the command is distributed, committed, then applied.

We also need to decide how peers communicate:

1. Long connections - easier to maintain state, can guarantee message order within a connection, but a slow message blocks others
2. Short connections - simpler to write, but need to handle message ordering with sequence numbers. Network disconnections may create many connections.
3. Connection pools - also need message ordering, but can configure different concurrency for different messages. Higher complexity.

## Goals

- Election (2A)
- Log distribution (2B)
- Persistence (2C)
- Snapshot (2D)

## Election

Election is the process where peers elect a leader. After election, the leader distributes log data to other peers.

Raft's election is actually quite similar to real life - it maps to student body elections:

- 3 peers prepare for election --- 3 students start campaigning for class leader
- Only the highest term is valid --- we want the latest election results
- Majority agreement is sufficient --- need to get majority votes
- Agreement condition is that the candidate's log count is higher --- agreement condition is that the candidate's experience (log) is more complete than yours

### Details

- Term handling during the vote process
- How peers vote

### Pseudocode

```python
// For each Raft object, we need to periodically check state and conduct elections.

while not_killed:

	if state == follower and recv_no_heart_beat > ELECTION_TIMEOUT:

		send_vote_request_to_all
```

```python
// We also need to handle other peers' votes

if has_more_logs:

	return vote_granted

return vote_fail
```

```python
// After receiving vote replies

save_vote_reply_for_this_peer

check_if_being_leader_now
```

### Detail Handling {#raft-detail}

1. When sending and receiving votes, ideally in the same term. But in practice they won't be. Ignore or reject lower-term messages. For high-term VoteRequests, first raise your own term, clear your collected votes, then process.
2. Each peer cannot vote for two candidates in the same term. (Can abstain and switch - see condition 3a)
3. For the same term, each peer has multiple voting strategies, similar to real life:
    1. Candidates vote for themselves first, then campaign. Enhancement: during voting, if collected votes prove you can't win, abstain early and vote for others
    2. Campaign first, cast self-vote at the last stage (on my machine, this strategy is about 10% faster)
4. In candidate state, configure a timer for election. If a normal leader encounters a rogue VoteRequest, it can fall back to follower and start election in the next heartbeat. For other followers receiving rogue VoteRequests, they can choose not to process them (see point 2).



## Log Distribution

After a peer becomes leader, it can distribute data in the current term.

### Details

- When to persist data - immediately upon receipt or batch processing?
- How to quickly update_index, how to give peers the latest data after a few rounds of negotiation
- Raft's paper says only commit data from the current term, but what if the last data is from a previous term with no new data coming? Can it be committed?
- How to apply after data is committed

### Pseudocode

```bash
// Start receiving data

while not_killed:

	recv_a_log

	send_to_peers
```

```bash
// Receive peer feedback

update_log_indexes

update_commit_info
```

```python
// At this stage, all peers need to apply data

while not_killed:

	get_last_applied_msg

	do_apply
```

### Detail Handling {#when-to-save}

- Persist immediately upon receipt, otherwise tests may pass once or twice but fail 1-2 times out of 10. When persisting, consider whether to store just raft logs or include snapshots. Note that serialization is needed during persistence, so kvraft tests on low-performance machines like my Raspberry Pi may fail.
- How to quickly update index - depends on whether AppendEntriesReply has a conflict:
    - If no conflict:
        - Update the peer's last item (log_index + term)
        - Collect all peers' last items. If in current term, determine majority position to update commit pos
        - If all peers' last items are from an old term, take minimum index across all peers to update commit pos
    - If conflict (conflict_log_idx, conflict_log_term):
        - If conflict_log_term matches the current conflict_log_idx, set next_match = conflict_log_idx + 1
        - If not matching, next_match = conflict_log_idx, or configure next_match = conflict_log_idx - N (N can be customized, e.g., half the current log count), or configure based on term info
- After data is committed, we can apply. My initial approach was:

    ```python
    for last_applied < commited:

    	do_apply(logs[last_applied])

    	last_applied++
    ```

    But this may have lock issues. Also, if the log contains snapshot messages, the log may be truncated. Since the log itself is immutable, I prefer the following approach with better lock granularity:

    ```go
    cmd, ok := get_to_apply(last_applied)

    do_apply(cmd)

    last_appied = cmd.idx
    ```


## Persistence

In tests and production, raft data can become invalid at any time. We need to ensure that Snapshot + LogApply eventually reaches consistency, so storage must include LogState + SnapState. But in this test, there's no Snap. We only need to store all logs.

### Details

- What state to store? The Raft paper shows only these - do we need more?

![Untitled](/raft/raft_persist.png)

### Pseudocode

```python
// when killed
saved_data = rf.Data.Encode() // term, vote, log

// when restore
rf.Data.Decode(saved_data)
```

### Detail Implementation

- If logs don't start from 0, also need EntryIdx, EntryTerm, SnapIdx, SnapTerm.

## Snapshot

Snapshot ensures that Snapshot + (SnapIdx ... CommitIdx).apply() produces the same result as (0 .. CommitIdx).apply()

### Details

- When to generate snapshots
- When to send snapshots
- What to do when receiving a snapshot

### Pseudocode

```python
// Receive Snap request - snap_data needs to be provided by external state machine
persist(snap_data, snap_idx)
clear_useless_logs()

// When sending heartbeat, if the peer's needed log no longer exists locally
if log_not_exists:
	send_snap_data()

// When shutting down
persist(snap_data, raft_data)
```

### Detail Handling

- Raft can expose its state externally, e.g., current log size, so external systems can take snapshots at any time. When taking a snapshot, the external system passes in its applied idx and latest state. The Raft service then needs to store:
    - SnapState, SnapIdx, SnapTerm (optional)
    - Logs not applied

    You don't have to remove all applied logs - you can keep some.

- Continuing from above, if all applied logs are removed but not distributed to other peers (less than 1/2), then we can only send snapshots.
- After receiving a snapshot, persist first - same logic as commit.



## Summary {#avoid-lock}

This project is the foundation for others. Many tech decisions were gradually refined while working on subsequent projects.

When interacting with external systems, this module should maintain minimal interfaces:

- Start / Kill
- Snapshot
- OnMessage

If external systems want more Raft data, they can pass control commands like RaftState{}.

Raft can fill in internal information before passing this to the state machine, producing:

RaftState {

     LogInfo:

     SnapInfo:

     TotalIdx:

}

Raft can also proactively communicate externally, sending state changes like BecomingALeader, etc.

These approaches help avoid various deadlock issues.

[Back](/raft/)
