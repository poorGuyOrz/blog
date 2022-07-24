---
title: "Raft"
date: 2022-05-30T18:51:02+08:00
draft: false
tags: ["论文", "Raft", "6.824"]
---

> * [译文](https://college.blog.csdn.net/article/details/53671783?spm=1001.2101.3001.6650.4&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-4-53671783-blog-42742105.pc_relevant_paycolumn_v3&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-4-53671783-blog-42742105.pc_relevant_paycolumn_v3&utm_relevant_index=8)
> * [原文](https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf)
> * [有用的飞书文档](https://hardcore.feishu.cn/docs/doccnMRVFcMWn1zsEYBrbsDf8De#)

* 和其他的算法相比
  1. Strong leader  
    日志只能从领导者发送到其他节点  
  2. Leader election  
    随机计时器选举领导，在心跳机制上加上一些额外的工作  
  3. Membership changes  
    角色变换

## Replicated state machines  

复制状态机一般基于日志实现，通俗的理解只要所有的机器按照相同的顺序执行指令，那么每个节点的状态都是确定的，所以需要把指令日志复制到其他节点上去，这就是一致性算法的工作  

如果只是要求最终所有的节点都执行一样顺序的指令，而不要求实时性，则可以限定
1. 只有一个节点可以进行写操作，因为只有写操作才可以改变系统的状态
2. 写节点同步指令到其他节点，最终所有节点指令顺序一致

一致性算法的共有特性
  * 安全性  
    不会返回一个错误结果，只要是在非拜占庭错误情况下，包括网络延迟，乱序，丢包，分区，冗余等都可以保障正确性
  * 可用性  
    集群只要大多数机器可以正常通信，就可以确保可用，失败节点可以忽略或者后续恢复状态，大多数指的是半数以上
  * 不依赖时序保证一致性  
    时钟错误或者消息延迟只有在极端情况下才会导致可用性
  * 慢节点不会影响消息的反馈，消息可以快速的响应

> [拜占庭将军问题](https://liebing.org.cn/2020/02/14/byzantine_generals_problem/)

## Paxos

1. 难以理解
2. 没有公认的可以实现的基础架构，大部分系统从Paxos开始，在遇到问题的时候自行想办法解决，导致最后的系统实现只能是类似Paxos的算法

## Raft

管理复制状态机的一种算法，他会在集群中选举一个leader，之后会复制所有的日志到其他节点实现一致性  

他可以分解为三个问题  
1. 领导选举
    一个新的领导人需要被选举出来，当先存的领导人宕机的时候  
2. 日志复制  
  领导人必须从客户端接收日志然后复制到集群中的其他节点，并且强制要求其他节点的日志保持和自己相同  
3. 安全性  
  在 Raft 中安全性的关键是在图 3 中展示的状态机安全：如果有任何的服务器节点已经应用了一个确定的日志条目到它的状态机中，那么其他服务器节点不能在同一个日志索引位置应用一个不同的指令  

可以在这个[网站](http://thesecretlivesofdata.com/raft/)查看实例
* 节点有三种状态
  1. Follower  
  2. Candidate  
  3. Leader   

他们之间的转换关系如下图
![rr](/posts/course/6.824/images/raft/role.png)



任期在 Raft 算法中充当逻辑时钟的作用，这会允许服务器节点查明一些过期的信息比如陈旧的领导者。每一个节点存储一个当前任期号，这一编号在整个时期内单调的增长。当服务器之间通信的时候会交换当前任期号；如果一个服务器的当前任期号比其他人小，那么他会更新自己的编号到较大的编号值。如果一个候选人或者领导者发现自己的任期号过期了，那么他会立即恢复成跟随者状态。如果一个节点接收到一个包含过期的任期号的请求，那么他会直接拒绝这个请求。

下面是详细的细节参数
![state](/posts/course/6.824/images/raft/state.png)

下面的参数要求在节点上持久存在  
* `currentTerm`  
  服务器最后一次知道的任期号，初始化为 0，持续递增
* `votedFor`  
  当前获得选票的候选人的Id
* `log[]`  
  日志条目集；每一个条目包含一个用户状态机执行的指令，和收到时的任期号

下面的参数在节点上是随时变化的
* `commitIndex`  
  已知的最大的已经被提交的日志条目的索引值
* `lastApplied`  
  最后被应用到状态机的日志条目索引值，初始化为 0，持续递增

下面的参数需要在leader重新选举之后变化的
* `nextIndex[]`  
  对于每一个服务器，需要发送给他的下一个日志条目的索引值，初始化为领导人最后索引值加一  
* `matchIndex[]`  
  对于每一个服务器，已经复制给他的日志的最高索引值  

![AppendEntries RPC](/posts/course/6.824/images/raft/rpc.png)

这篇图表示的是rpc的参数信息以及返回值，由领导人负责调用来复制日志指令；也会用作heartbeat
参数  
* `term`  
  领导人的任期号
* `leaderId`  
  领导人的 Id，以便于跟随者重定向请求
* `prevLogIndex`  
  新的日志条目紧随之前的索引值
* `prevLogTerm`  
  prevLogIndex 条目的任期号
* `entries[]`  
  准备存储的日志条目，表示心跳时为空；一次性发送多个是为了提高效率
* `leaderCommit`  
  领导人已经提交的日志的索引值

返回值  
* `term`  
  当前的任期号，用于领导人去更新自己
* `success`  
  跟随者包含了匹配上 prevLogIndex 和 prevLogTerm 的日志时为真


1. 如果 term < currentTerm 就返回 false  
2. 如果日志在 prevLogIndex 位置处的日志条目的任期号和 prevLogTerm 不匹配，则返回 false  
3. 如果已经已经存在的日志条目和新的产生冲突（相同偏移量但是任期号不同），删除这一条和之后所有的  
4. 附加任何在已有的日志中不存在的条目  
5. 如果 leaderCommit > commitIndex，令 commitIndex 等于 leaderCommit 和 新日志条目索引值中较小的一个  

![RequestVote RPC](/posts/course/6.824/images/raft/rvoteprc.png)

由候选人负责调用用来征集选票  

参数  
* `term`  
  候选人的任期号
* `candidateId`  
  请求选票的候选人的 Id
* `lastLogIndex`  
  候选人的最后日志条目的索引值
* `lastLogTerm`  
  候选人最后日志条目的任期号

返回值  
* `term`  
  当前任期号，以便于候选人去更新自己的任期号
* `voteGranted`  
  候选人赢得了此张选票时为真

1. 如果term < currentTerm返回 false  
2. 如果 votedFor 为空或者就是 candidateId，并且候选人的日志也自己一样新，那么就投票给他  

![Rules for Servers](/posts/course/6.824/images/raft/serverr.png)

上图说明了所有服务器的规则  
all node   
* 如果commitIndex > lastApplied，那么就 lastApplied 加一，并把log[lastApplied]应用到状态机中  
* 如果接收到的 RPC 请求中，任期号T > currentTerm，那么就令 currentTerm 等于 T，并切换状态为跟随者  

Follower   
* 响应来自候选人和领导者的请求  
* 如果在超过选举超时时间的情况之前都没有收到领导人的心跳，或者是候选人请求投票的，就自己变成候选人  

Candidate  

* 在转变成候选人后就立即开始选举过程
  * 自增当前的任期号(currentTerm)
  * 给自己投票 
  * 重置选举超时计时器
  * 发送请求投票的 RPC 给其他所有服务器
* 如果接收到大多数服务器的选票，那么就变成领导人  
* 如果接收到来自新的领导人的附加日志 RPC，转变成跟随者  
* 如果选举过程超时，再次发起一轮选举  

Leader  
* 一旦成为领导人：发送空的附加日志 RPC（心跳）给其他所有的服务器；在一定的空余时间之后不停的重复发送，以阻止跟随者超时
* 如果接收到来自客户端的请求：附加条目到本地日志中，在条目被应用到状态机后响应客户端
* 如果对于一个跟随者，最后日志条目的索引值大于等于 nextIndex，那么：发送从 nextIndex 开始的所有日志条目
  * 如果成功：更新相应跟随者的 nextIndex 和 matchIndex
  * 如果因为日志不一致而失败，减少 nextIndex 重试
* 如果存在一个满足N > commitIndex的 N，并且大多数的matchIndex[i] ≥ N成立，并且log[N].term == currentTerm成立，那么令 commitIndex 等于这个 N

> 上面的是关于实现raft的关键参数总结

raft必须保证下面的特性
* Election Safety  
  at most one leader can be elected in a given term
* Leader Append-Only  
  a leader never overwrites or deletes entries in its log; it only appends new entries
* Log Matching  
  如果两个日志在相同的索引位置的日志条目的任期号相同，那么我们就认为这个日志从头到这个索引位置之间全部完全
* Leader Completeness  
  如果某个日志条目在某个任期号中已经被提交，那么这个条目必然出现在更大任期号的所有领导人中
* State Machine Safety  
  如果一个领导人已经在给定的索引值位置的日志条目应用到状态机中，那么其他任何的服务器在这个索引位置不会提交一个不同的日志

基础的raft算法只需要两种基础的RPC，即上面提到的`RequestVote RPC`和`AppendEntries RPC`


### Leader Election
* 所有的节点初始为Follower  
* 如果Follower一段时间没有收到Leader的心跳包，此时Follower变为Candidate  
* Candidate向所有节点发起投票，Candidate遵循前面提到的基础规则，他在投票期间会得到三种结果
  1. 赢得选举，此时他获得半数投票，规定Follower在一轮选举中只能有一张选票，此时一定能确保只有一个node得到半数以上选票。此时leader会发送心跳包来停止投票过程
  2. 投票过程中，收到leader的心跳包，此时校验如果leader的任期号大于等于自己的任期好，则自己成为follower
  3. 没有人成为leader，此时在`election timeout `


在上述过程中，由一种超时类型  
* election timeout  
  1. 指的是Follower变更为Candidate的时间，不是固定时间，是在预设范围内的随机值，(150ms ~ 300ms)，超过时间，则Follower变为Candidate，此时发起一轮投票  
  2. 此时其他节点投票，且刷新超时时间

Leader存在的时候，所有的变动由Leader发起  

### Log Replication

* 每个变动作为一个node上log的条目  
* Leader记录未提交  
* 先向其他node同步记录  
* node收到变更消息之后，向Leader回复消息，但是不提交    
* 半数node回复消息之后，Leader commit记录  
* 此时Leader通知Follower提交记录  
* Follower commit  

Raft 算法保证所有已提交的日志条目都是持久化的并且最终会被所有可用的状态机执行
leader 记录最大的提交的日志的index，且每次rpc附带此消息，其他node使用此消息检验日志状态，

* 如果在不同的日志中的两个条目拥有相同的索引和任期号，那么他们存储了相同的指令。  
* 如果在不同的日志中的两个条目拥有相同的索引和任期号，那么他们之前的所有日志条目也全部相同。  


要使得跟随者的日志进入和自己一致的状态，领导人必须找到最后两者达成一致的地方，然后删除从那个点之后的所有日志条目，发送自己的日志给跟随者。所有的这些操作都在进行附加日志 RPCs 的一致性检查时完成。领导人针对每一个跟随者维护了一个 nextIndex，这表示下一个需要发送给跟随者的日志条目的索引地址。当一个领导人刚获得权力的时候，他初始化所有的 nextIndex 值为自己的最后一条日志的index加1（图 7 中的 11）。如果一个跟随者的日志和领导人不一致，那么在下一次的附加日志 RPC 时的一致性检查就会失败。在被跟随者拒绝之后，领导人就会减小 nextIndex 值并进行重试。最终 nextIndex 会在某个位置使得领导人和跟随者的日志达成一致。当这种情况发生，附加日志 RPC 就会成功，这时就会把跟随者冲突的日志条目全部删除并且加上领导人的日志。一旦附加日志 RPC 成功，那么跟随者的日志就会和领导人保持一致，并且在接下来的任期里一直继续保持。


此时系统又达到一致的状态
在Leader发送消息的时候，设置超时类型为
* heartbeat timeout


## 安全性

### 选举限制

日志只从领导人传给跟随者，并且领导人从不会覆盖本地日志中已经存在的条目
投票规则
2. 如果 votedFor 为空或者就是 candidateId，并且候选人的日志也自己一样新，那么就投票给他   
RPC 中包含了候选人的日志信息，然后投票人会拒绝掉那些日志没有自己新的投票请求  
Raft 通过比较两份日志中最后一条日志条目的索引值和任期号定义谁的日志比较新。如果两份日志最后的条目的任期号不同，那么任期号大的日志更加新。如果两份日志最后的条目任期号相同，那么日志比较长的那个就更加新  

![选举安全性](/posts/course/6.824/images/raft/pc1.png)

使用规则
> 1. 如果term < currentTerm返回 false  
> 2. 如果 votedFor 为空或者就是 candidateId，并且候选人的日志也自己一样新，那么就投票给他  
* 只会投票给任期比自己大，且日志比自己新  
* 提交之前，确保半数节点已经和leader日志状态一致

1. 图a s1是leader，日志只写s1和s2之后宕机
2. 图b s5得到s3,s4,s5的投票成为leader，写日志3，然后宕机
3. 图c s1上线通过s1,s2,s3,s4成为leader，同时复制之前的日志到s3，且写日志s4，宕机
4. 图d s5通过s2,s3,s4,s5成为leader，复制log到所有节点，此时覆盖之前部分日志，理论上是个错误行为
5. 限定提交日志的定义为半数节点收到日志，此时即使失败了，后续不会发生s5成为leader的情况

提交的日志是安全的，不会被删除覆盖

### 论证

核心是所有的Candidate选举成功的前提是获得半数选票  
那么此时他的日志是最新的，在成功之后会同步日志  
反之，如果有未提交的日志，则他不会选举成功

* follower和Candidate宕机  
  此时选择无限次重试，由于幂等性设计，之后上线接受日志的时候，是不会有重复日志的情况产生的，
* 时间要求  
  广播时间（broadcastTime） << 选举超时时间（electionTimeout） << 平均故障间隔时间（MTBF）
* 网络分区的时候，大多数节点的分区会重新选取leader，少数分区的那部分没有leader，会一致陷入选举的过程中，避免的方法是选举的时候先预发送一个选举RPC，确定自己在多数分区，才进行选举

## 成员变更
https://blog.openacid.com/distributed/raft-bug/

* 新节点设置保护时间，只复制日志，不参与选举
* 


## 日志压缩

使用快照压缩记录，快照记录lastIncludedIndex和lastIncludedTerm，各个节点单独建立快照，快照建立之后，之前的日志可以清楚

通常节点之间不会传递快照，但是加入新加入节点或者某节点落后许多的时候，可以使用快照让节点状态快速达到最新，此时定义InstallSnapshot

![InstallSnapshot RPC](/posts/course/6.824/images/raft/pc2.png)

接收者使用上述信息决定快照的处理方式  
* 如果快照比节点的日志新，则丢弃所有日志，直接使用此快照  
* 如果是快照在节点日志之内，则快照之前的日志清楚，保留快照以及之后的日志

快照只会从leader发送

* 快照会导致性能问题
  * 设置日志阈值，定期设置快照，不可太快或者太慢，太快浪费资源，太慢会导致日志太大
  * 写时复制，避免快照写入时导致的性能问题

## 客户端

* 客户端之和leader交互，最开始时随机选择节点，follower节点拒绝请求且 回复leader节点编号，之后客户端在使用此信息和leader建立链接，leader失败之后，重复此过程  
* 如果客户端发送指令之后leader崩溃而没有回复，则之后client会重试此消息，此时对client的消息设置序列号，之后通信的时候校监序列号，如果序列号指令被应用，则下次leader直接回复client  
*  读请求可能发生在慢节点上，此时使用lastApplied来进行检测，客户端需要维护一个相同意义的值，和每次读取的节点上的lastApplied对比，客户端只会看见比自己的值大的值，因为每次log提交的时候，lastApplied会增加，所以读取的时候，必须确定每次都是新的，如果不是，则重新选择节点尝试，或者直接选择leader进行读取
