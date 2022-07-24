---
title: "6.824 LAB"
date: 2022-05-28T16:32:04+08:00
draft: true
---

## Raft

Implement Raft leader election and heartbeats (AppendEntries RPCs with no log entries). The goal for Part 2A is for a single leader to be elected, for the leader to remain the leader if there are no failures, and for a new leader to take over if the old leader fails or if packets to/from the old leader are lost. Run go test -run 2A to test your 2A code.
实现Raft领导选举和心跳(无日志条目的附录 rpc)。2A的目标是选出一个领导人，如果没有失败，领导人仍然是领导人，如果旧领导人失败，或者与旧领导人之间的信息丢失，则由新领导人接管。运行 go test-Run 2A 来测试你的2a 代码。

* 参考图2，关注sending and receiving RequestVote RPCs以及node参与选举的规则  
Follow the paper's Figure 2. At this point you care about sending and receiving RequestVote RPCs, the Rules for Servers that relate to elections, and the State related to leader election,

* 添加state for leader election，定义log相关  
Add the Figure 2 state for leader election to the Raft struct in raft.go. You'll also need to define a struct to hold information about each log entry.

* 补全RequestVoteArgs和RequestVoteReply，修改Make()，注意RequestVote  
Fill in the RequestVoteArgs and RequestVoteReply structs. Modify Make() to create a background goroutine that will kick off leader election periodically by sending out RequestVote RPCs when it hasn't heard from another peer for a while. This way a peer will learn who is the leader, if there is already a leader, or become the leader itself. Implement the RequestVote() RPC handler so that servers will vote for one another.

* 修改AppendEntries ，用于心跳机制，
To implement heartbeats, define an AppendEntries RPC struct (though you may not need all the arguments yet), and have the leader send them out periodically. Write an AppendEntries RPC handler method that resets the election timeout so that other servers don't step forward as leaders when one has already been elected.

* 随机超时时间  
Make sure the election timeouts in different peers don't always fire at the same time, or else all peers will vote only for themselves and no one will become the leader.

* 测试要求心跳间隔在10/s以下  
The tester requires that the leader send heartbeat RPCs no more than ten times per second.

* tester要求leader在集群可用的情况下5s内选举  
The tester requires your Raft to elect a new leader within five seconds of the failure of the old leader (if a majority of peers can still communicate). Remember, however, that leader election may require multiple rounds in case of a split vote (which can happen if packets are lost or if candidates unluckily choose the same random backoff times). You must pick election timeouts (and thus heartbeat intervals) that are short enough that it's very likely that an election will complete in less than five seconds even if it requires multiple rounds.

* 论文时间建议在150~300ms，上面有10/s的限制，所以时间需要调整的比论文中的稍微大一些   
The paper's Section 5.2 mentions election timeouts in the range of 150 to 300 milliseconds. Such a range only makes sense if the leader sends heartbeats considerably more often than once per 150 milliseconds. Because the tester limits you to 10 heartbeats per second, you will have to use an election timeout larger than the paper's 150 to 300 milliseconds, but not too large, because then you may fail to elect a leader within five seconds.
You may find Go's rand useful.

You'll need to write code that takes actions periodically or after delays in time. The easiest way to do this is to create a goroutine with a loop that calls time.Sleep(); (see the ticker() goroutine that Make() creates for this purpose). Don't use Go's time.Timer or time.Ticker, which are difficult to use correctly.
The Guidance page has some tips on how to develop and debug your code.
If your code has trouble passing the tests, read the paper's Figure 2 again; the full logic for leader election is spread over multiple parts of the figure.
Don't forget to implement `GetState()`.
The tester calls your Raft's rf.Kill() when it is permanently shutting down an instance. You can check whether Kill() has been called using rf.killed(). You may want to do this in all loops, to avoid having dead Raft instances print confusing messages.
Go RPC sends only struct fields whose names start with capital letters. Sub-structures must also have capitalized field names (e.g. fields of log records in an array). The labgob package will warn you about this; don't ignore the warnings.

