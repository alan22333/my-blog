---
title: raft
description: 'mit6824 lab3记录'
tags: ['raft']
toc: false
date: 2026-01-28 21:49:20
categories:
    - go
    - project
---

## Part-A: 选主

### 核心内容

> 这一章要解决的核心问题是：在一个去中心化的系统里，当老大（Leader）挂了，群众（Followers）如何自动选出一个新老大，并且保证只选出一个？

#### 节点角色

1. **Follower（跟随者/群众）**：

  - **特点**：被动。它只响应来自 Leader 和 Candidate 的请求。

  - **行为**：所有节点启动时都是 Follower。如果听不到 Leader 的消息（心跳超时），它就会变成 Candidate。

2. **Candidate（候选人/竞选者）**：

  - **特点**：主动拉票。

  - **行为**：它会给自己投票，并发消息给其他人求票。如果赢得大多数选票，晋升为 Leader。

3. **Leader（领导者）**：

  - **特点**：绝对权威。处理所有客户端请求。

  - **行为**：不断给 Follower 发送心跳（Heartbeat），告诉大家“我还活着，不要造反”。

#### 选举机制

**1. 心跳与超时**

Raft 的选举完全依靠**时间**来驱动。这里有两个至关重要的时间概念，请务必区分清楚：

1. **心跳间隔 (Heartbeat Interval)**：
  - **谁发？** Leader。
  - **频率**：很高（例如每 50ms - 100ms）。
  - **作用**：Leader 不断给所有 Follower/Candidate 发空消息(AppendRpc)，收到的节点需要保持或切换为“Follower”。

2. **选举超时 (Election Timeout)**：
  - **谁用？** Follower。
  - **时长**：随机的（例如 150ms - 300ms 之间）。
  - **作用**：Follower 进程中持有一个倒计时器，每次收到 Leader 的心跳，倒计时器就**清零重置**。
  - **触发**：如果倒计时归零了，Follower 就会认为 Leader 挂了，立马造反，切换为Candidate触发选举。

#### 选举流程

1. 正常状态。老leader不断发送心跳（空AppendRpc），Follower和已经变成Candidate的Follower收到立即保持Follower（即使已经有节点投给他票了，也要立即作为普通节点）；
2. 领导下线。老leader宕机，停止心跳。
3. 触发超时。Follower切换到Candidate，增加自己任期，给自己头上一票先。
4. 发起选举。Candidate广播发送RequestVoteRpc，其中说明自己自己的任期和其他一些信息（按下不表先）。
5. 群众投票。其他节点（包括Candidate），在投票有两个原则“唯任期论”和“先到先得”，都是顾名思义。
6. 统计结果。Candidate在处理Rpc结果的同时统计得票数，超过半数立即自封Leader，不足则继续尝试，直到成功或收到心跳（已经有其他人上位成功了，变回跟随着）。

### 实现思路

> 一些关键的代码实现细节总结

首先考虑这个机制下Raft节点需要维护的属性
```Go
	currentTerm int  // 当前任期，任期是raft算法中最重要的属性
	votedFor    int  // 投票情况，-1表示还没投
	state       State// 当前角色

	lastElectionReset time.Time // 上一次重制计时器的时间，用于计算超时发起选举
```

然后从流程出发，先看看正常运行下需要实现的东西，显然是“心跳机制”
在Raft算法中，一共只有两种Rpc：
Raft 节点之间只通过两种主要的消息（RPC - 远程过程调用）进行沟通：

1. **RequestVote RPC**：用于**选举**。
  - Candidate：“请投我一票！”

2. **AppendEntries RPC**：用于**日志复制**和**心跳**。
  - Leader：“这是新数据，记下来。” 或者 “我还活着（不带数据）。”

所以我们先实现AppendEntries RPC：
> 不要忘记RPC属性大写
```Go
// 根据论文Fig2提示，选择目前需要的参数
type AppendEntriesArgs struct {
	Term    int  // 任期代表发送方的“实力”->任期比别人小，别人是不会理你的
	Leader  int  // 当前的领导
	Entries []any//日志，先不用管

	// prevLogIndex int
	// prevLogTerm  int
	// leaderCommit int
}

type AppendEntriesReply struct {
	Term    int  // 回复自己的最新任期
	Success bool // 接不接受（复制or心跳）
}

func (rf *Raft) AppendEntries(args *AppendEntriesArgs, reply *AppendEntriesReply) {
	rf.mu.Lock()
	defer rf.mu.Unlock()

	// 先判断是否认可，看看实力（任期）
	if args.Term < rf.currentTerm {
		reply.Term = rf.currentTerm
		reply.Success = false
		return
	}

	// 同步任期
	rf.currentTerm = args.Term

	// 重制倒计时
	rf.lastElectionReset = time.Now()

    // 听到圣旨了，老老实实做跟随者
	rf.state = StateFollower

	reply.Term = rf.currentTerm
	reply.Success = true

}

// 调用
func (rf *Raft) sendAppendEntries(server int, args *AppendEntriesArgs, reply *AppendEntriesReply) bool {
	ok := rf.peers[server].Call("Raft.AppendEntries", args, reply)
	return ok
}

```

然后我们想到什么时候发送心跳Rpc呢？显然是leader的日常工作（循环），找到节点的ticker：
为了看起来清晰一点，就去掉一些并发锁之类的，这当然是非常重要的，这里先不讨论（我自己感觉写的也不咋地）
```Go
func (rf *Raft) ticker() {
    // 主循环
	for rf.killed() == false {

		if rf.state == StateLeader {
            // 是领导
			rf.broadcastHeartbeat()
		}else if time.Since(rf.lastElectionReset) > timeout {
            // 不是领导，检查有没有超时（这个timeout是一个合理的随机值，保证快速选出减少平票）
			rf.startElection()
		}

        // 心跳间隔
		time.Sleep(50 * time.Millisecond)
	}
}
```

然后我们考虑一下怎么广播心跳信号，1.构造一个“当前状态”的AppendRpc，2.目标是除自己外的所有节点，3.并发发送，4.处理回收（看看有没有真领导，即任期更大的节点，有的话直接降级为Follower跟着真领导混）：
```Go
func (rf *Raft) broadcastHeartbeat() {

    // 在并发环境中，状态瞬息万变，再次确认自己的状态
	if rf.state != StateLeader {
		return
	}

	for peer := range rf.peers {
		if peer == rf.me {
			continue
		}
		go func(server int) {
            // 构造Rpc
			args := &AppendEntriesArgs{
				Term:    rf.currentTerm,
				Leader:  rf.me,
				Entries: nil,
			}
			reply := &AppendEntriesReply{}
			if rf.sendAppendEntries(peer, args, reply) {
                // 处理回复

                // 同理，说不定办完事自己已经是普通节点
                if rf.state != StateLeader {
                    return
                }

                // 退位操作（回到普通Follower）
                // 更新任期、更新状态、更新投票状态、更新计时器
				if reply.Term > rf.currentTerm {
					rf.currentTerm = reply.Term
					rf.state = StateFollower
					rf.votedFor = -1
					rf.lastElectionReset = time.Now()
				}
			}
		}(peer)
	}
}
```

那么显然这个功能只剩下最重要的“选举”环节了，上文我们已经定义了触发时机（超时），现在我们定义一下这个Rpc：
```Go
type RequestVoteArgs struct {
	Term        int
	CandidateId int
}

type RequestVoteReply struct {
	Term        int
	VoteGranted bool
}

func (rf *Raft) RequestVote(args *RequestVoteArgs, reply *RequestVoteReply) {

    // 1. 检查任期
	if args.Term > rf.currentTerm {
		rf.currentTerm = args.Term
		rf.state = StateFollower
		rf.votedFor = -1
	}

    // 2. 赋值回复的 Term (确保是新的)
    reply.Term = rf.currentTerm

    // 3. 拒绝旧任期
	if args.Term < rf.currentTerm {
		return
	}

    // 4. 投票逻辑：“我还没投过” or “已经给你投过一次了”
	if rf.votedFor == -1 || rf.votedFor == args.CandidateId {
		reply.VoteGranted = true
		rf.votedFor = args.CandidateId
        // 投票也算活跃，重置计时器（这里我不太理解）
		rf.lastElectionReset = time.Now()
	}
}

```

有了Rpc就可以发起选举了：
```Go
// 发起选举
func (rf *Raft) startElection() {
	// 选举后要做的事：
	// 1. 成为候选人，开一个新任期
	// 2. 给自己投票 + 拉票
	// 3. 统计选票

    // 依旧并发环境检查状态
	if rf.state == StateLeader || time.Since(rf.lastElectionReset) < 10*time.Millisecond {
		return
	}

	// 1. 转换为 Candidate 状态，开启新任期
	rf.state = StateCandidate
	rf.currentTerm++

    // 2. 给自己先来一票
	rf.votedFor = rf.me
	votes := 1

	rf.lastElectionReset = time.Now() // 重置选举计时器

	for peer := range rf.peers {
		if peer == rf.me {
			continue
		}
	    // 2. 并发拉票
		go func(serverID int) {
			args := &RequestVoteArgs{
				Term:        rf.currentTerm,
				CandidateId: rf.me,
			}
			reply := &RequestVoteReply{}
			if rf.sendRequestVote(peer, args, reply) {

                //3. 处理一下选票
				// 依旧响应结束检查当前（任期、状态），这个很重要

                // 任期不足，退化
				if reply.Term > rf.currentTerm {
					rf.state = StateFollower
					rf.currentTerm = reply.Term
					rf.votedFor = -1
					return
				}

				// 可能我发请求时是 Candidate，收到回复时已经是 Follower 了
				if rf.state != StateCandidate || rf.currentTerm != args.Term {
					return
				}

				// 检查投票
				if reply.VoteGranted {
					votes++
					if votes > len(rf.peers)/2 {
                        // 登基！
						rf.state = StateLeader
					}
				}
			}
		}(peer)
	}
}
```

以上就是Lab3-A的核心内容了，其中的并发加锁解锁处理没有讨论，读者自己折腾吧
