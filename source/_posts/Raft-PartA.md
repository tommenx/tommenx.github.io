---
title: Raft-PartA
date: 2017-10-22 18:39:29
categories:
- 分布式系统
- MIT 6.824
tags:
- raft
- golang
---
本次的实验为MIT6.824 Distribute System 中 Lab2 raft part A,通过课程所给的代码的骨架，要求实验raft中选举部分的功能。由于我的水平较渣，觉得课程的难度较大，与其说是实验比如说是对Github上的大神的代码进行理解，并结合论文进行思考。希望有朝一日能够自己做出一个实验啊！

> Implement leader election and heartbeats (`AppendEntries` RPCs with no log entries). The goal for Part 2A is for a single leader to be elected, for the leader to remain the leader if there are no failures, and for a new leader to take over if the old leader fails or if packets to/from the old leader are lost. Run `go test -run 2A` to test your 2A code.

Part A 部分要求实现raft的选举功能，其中raft中的每一个server都会维护自己的一个raft的结构体，其中只有两种RPC：`AppendEntries RPC` 和 `RequestVote RPC`。其中当`AppendEntres RPC`中的log为空时用作`HeartBeat`维护server 和 follower之间的通信，而`RequestVote RPC`则用作当follower转变成candidate时，向其他的server请求投票时的通信。整个选举的过程可以概括为下面的这张图，而其中又有许多的细节需要结合论文中的Figure 2中的每一个条件实现；

{% asset_image election.png election%}
## 可以概括为如下几个步骤
1. 系统初始化时，所有的server都是FLLOWER状态,并维护一个计时器，计时结束会成为CANDIDATE并开始选举；FOLLOWER每当收到一次`Herat Beat`或`RequestVote RPC`时重置计时器；
2. CANDIDATE在选举的时候依旧会维护一个计时器向所有的SERVER发送`Request Vote`，在结束时仍未选举完，重新开始选举。当CANDIDATE获得大部分的投票后变为LEADER;在选举的过程中若收到来自更新的term的`AppendEntries RPC`或者自己的term小于其他server的term时，转变为FOLLOWER；
3. LEADER开始广播`Heart Beat`包，广播自己成为LEADER的事实，并且维护和FOLLOWER之间的通信。当发现FOLLOWER的reply中有更高的term时转变为FOLLOWER；

## 代码解析


### 定义一些常量
``` go
const (
	FOLLOWER = iota
	CANDIDATE
	LEADER

	HEARTBEAT_INTERVAL    = 100
	MIN_ELECTION_INTERVAL = 400
	MAX_ELECTION_INTERVAL = 500
)
```
### 创建必要的结构体
raft相当于server中的state machine
``` go
type Raft struct {
	mu        sync.Mutex          // Lock to protect shared access to this peer's state
	peers     []*labrpc.ClientEnd // RPC end points of all peers
	persister *Persister          // Object to hold this peer's persisted state
	me        int                 // this peer's index into peers[]

	// Your data here (2A, 2B, 2C).
	// Look at the paper's Figure 2 for a description of what
	// state a Raft server must maintain.

	votedFor     int   //投票给某个CANDIDATE id
	voteAcquired int   //收到的票数总数，用于判断能否成为LEADER
	state        int32 //server当前的状态
	currentTerm  int32 //server当前的term

	voteCh   chan struct{} //由于是并行发送用于指示server收到VoteRequest后的操作
	appendCh chan struct{} //收到Append Entries后的操作

	electionTimer *time.Timer //用于FOLLOWER和CANDIDATE的操作
}
```
发送`RequestVote RPC`包中的内容
```go
type RequestVoteArgs struct {
	Term        int32
	CandidateId int
}
```
server对于`RequestVote RPC`包的回复
```go
type RequestVoteReply struct {
	// Your data here (2A).
	Term        int32
	VoteGranted bool
}
```
发送`AppendEntries RPC`包中的内容
```go
//此处只涉及选举，不考虑replication及log中的内容
type AppendEntriesArgs struct {
	Term     int32
	LeaderID int
}
```
接收`AppendEntries RPC`包，并回复
同样这里不考虑CANDIDATE中的log是否是最新的，只要有合适的`VoteReuest PRC`就投票
```go
type AppendEntriesReply struct {
	Term    int32
	Success bool
}
```
###一些重要的工具函数


```go
func (rf *Raft) getTerm() int32 {
	return atomic.LoadInt32(&rf.currentTerm)
}

func (rf *Raft) incrementTerm() int32 {
	return atomic.AddInt32(&rf.currentTerm, 1)
}

func (rf *Raft) isState(state int32) bool {
	return atomic.LoadInt32(&rf.state) == state
}

func (rf *Raft) GetState() (int, bool) {

	var term int
	var isleader bool
	// Your code here (2A).
	term = int(rf.getTerm())
	isleader = rf.isState(LEADER)
	return term, isleader
}

func randElectionDuration() time.Duration {
	r := rand.New(rand.NewSource(time.Now().UnixNano()))
	return time.Millisecond * time.Duration(
    			r.Int63n(MAX_ELECTION_INTERVAL-MIN_ELECTION_INTERVAL)+MIN_ELECTION_INTERVAL
       	)
}
```

### CANDIDATE发送`RequestVote RPC`包
用于CANDIDATE发送包和对reply的处理

- 遍历每一个server，若投票了则voteGranted自增1
- 若发现reply中的term大于自身当前的term，则转变为FOLLOWER

```go
func (rf *Raft) broadcastVoteReq() {
	args := RequestVoteArgs{Term: atomic.LoadInt32(&rf.currentTerm), CandidateId: rf.me}
	for i, _ := range rf.peers {
		if i == rf.me {
			continue
		}
		// is it is candidate then send vote req
		go func(server int) {
			var reply RequestVoteReply
			if rf.isState(CANDIDATE) && rf.sendRequestVote(server, &args, &reply) {
				rf.mu.Lock()
				defer rf.mu.Unlock()
				//deal with the reply
				if reply.VoteGranted {
					rf.voteAcquired += 1
				} else {
					if reply.Term > rf.currentTerm {
						rf.currentTerm = reply.Term
						rf.updateStateTo(FOLLOWER)
					}
				}
			} else {
				fmt.Printf("Server %d send vote req failed.\n", rf.me)
			}
		}(i)
	}
}
```


### server对`RequestVote RPC`的处理
依据论文Figure 2中的要求，有以下几个处理的要点：
- 判断自身的term是否大于CANDIDATE的term，若是则拒绝投票，并更新reply中的term
- 若当前的term小于CANDIDATE发送的term，更新自身的term，并将自己的状态变为follower（当前状态为CANDIDATE或LEADER）
- 当两者的term相等时，必须是初始状态，否则拒绝投票（FOLLOWER转变为CANDIDATE时候term自增1，因此必须大于其他server）
- 投票成功后，利用信道通知主循环（整个操作是并行的）


```go
func (rf *Raft) RequestVote(args *RequestVoteArgs, reply *RequestVoteReply) {
	// Your code here (2A, 2B).
	rf.mu.Lock()
	defer rf.mu.Unlock()
	if args.Term < rf.currentTerm {
		reply.VoteGranted = false
		reply.Term = rf.currentTerm
	} else if args.Term > rf.currentTerm {
		rf.currentTerm = args.Term
		rf.updateStateTo(FOLLOWER)
		rf.votedFor = args.CandidateId
		reply.VoteGranted = true
	} else {
		if rf.votedFor == -1 {
			rf.votedFor = args.CandidateId
			reply.VoteGranted = true
		} else {
			reply.VoteGranted = false
		}
	}
	if reply.VoteGranted == true {
		go func() { rf.voteCh <- struct{}{} }()
	}
}
```


### LEADER广播`AppendEntries RPC`，和对reply的处理

由于这里只是单纯的选举所以处理的逻辑很简单，只处理失败的回复：

- 如果reply中的term大于当前LEADER中的term，则LEADER将自己的状态变成FOLLOWER


```go
func (rf *Raft) broadcastAppendEnrties() {
	args := AppendEntriesArgs{Term: atomic.LoadInt32(&rf.currentTerm), LeaderID: rf.me}
	for i, _ := range rf.peers {
		if i == rf.me {
			continue
		}
		go func(server int) {
			var reply AppendEntriesReply
			if rf.isState(LEADER) && rf.sendAppendEntries(server, &args, &reply) {
				rf.mu.Lock()
				defer rf.mu.Unlock()
				if reply.Success {

				} else {
					if reply.Term > rf.currentTerm {
						rf.currentTerm = reply.Term
						rf.updateStateTo(FOLLOWER)
					}
				}
			}
		}(i)
	}
}
```

### server对`AppendEntries RPC`的处理
同样，raft中的term可以作为判断是不是最新的依据，根据Figure 2，server在处理`AppendEntries RPC`，有如下注意点：
- 如果`args.Term < rf.currentTerm`，`reply.Term = rf.currentTerm`;
- 如果`args.Term > rf.currentTerm`，更新当前的term并且将状态改变为FOLLOWER（针对当前状态为CANDIDATE的server）


```go
func (rf *Raft) AppendEntries(args *AppendEntriesArgs, reply *AppendEntriesReply) {
	rf.mu.Lock()
	defer rf.mu.Unlock()
	//deal with the msg and reply
	if args.Term < rf.currentTerm {
		reply.Success = false
		reply.Term = rf.currentTerm
	} else if args.Term > rf.currentTerm {
		rf.currentTerm = args.Term
		rf.updateStateTo(FOLLOWER)
		reply.Success = true
	} else {
		reply.Success = true
	}
	go func() {
		rf.appendCh <- struct{}{}
	}()
}
```


### 状态转变的函数

需要注意的是当转变成FOLLOWER时，要将自己的`VoteFor`初始化为-1，当转变成CANDIDATE时候要注意开始选举LEADER
```go
func (rf *Raft) updateStateTo(state int32) {
	if rf.isState(state) {
		return
	}
	stateDesc := []string{"FOLLOWER", "CANDIDATE", "LEADER"}
	preState := rf.state
	switch state {
	case FOLLOWER:
		rf.state = FOLLOWER
		rf.votedFor = -1
	case CANDIDATE:
		rf.state = CANDIDATE
		rf.startElection()
	case LEADER:
		rf.state = LEADER
	default:
		fmt.Printf("Warning: invalid state %d, do nothing.\n", state)
	}
	fmt.Printf("In term %d: Server %d transfer from %s to %s\n",
		rf.currentTerm, rf.me, stateDesc[preState], stateDesc[rf.state])

}
```
### 开始新一轮选举的函数
按照论文的要求在开始选举新的LEADER的时候要重置计时器，因为CANDIDATE会为自己投票所以票数初始置为1
```go
func (rf *Raft) startElection() {
	rf.incrementTerm()  //first increase current term
	rf.votedFor = rf.me //vote for self
	rf.voteAcquired = 1 //acquire self's vote
	rf.electionTimer.Reset(randElectionDuration())
	rf.broadcastVoteReq()
}
```

#### 整个状态的循环实现
不断的循环，并根据当前server的角色进行不同的操作


#### FOLLOWER


- 当收到CANDIDATE发送的`VoteRequest RPC`时，重置计时器
- 当收到LEADER发送的`AppendEntries RPC`时，重置计时器
- 当计时器结束时，说明当前的server断绝联系，于是FOLLOWER转变为CANDIDATE开始新一轮的选举


#### CANDIDATE
- 在收到`AppendEntries RPC`时，说明LEADER依旧存在，停止选举转变为FOLLOWER
- 当计时器结束时，重新开始选举
- 当投票数大于总数的一半时，当选LEADER


#### LEADER
- 间隔一段时间发送`HeartBeat`心跳包维持LEADER和其他的server的通信


```go
func (rf *Raft) startLoop() {
	rf.electionTimer = time.NewTimer(randElectionDuration())
	for {
		switch atomic.LoadInt32(&rf.state) {
		case FOLLOWER:
			select {
			case <-rf.voteCh:
				rf.electionTimer.Reset(randElectionDuration())
			case <-rf.appendCh:
				rf.electionTimer.Reset(randElectionDuration())
			case <-rf.electionTimer.C:
				//time out
				rf.mu.Lock()
				rf.updateStateTo(CANDIDATE)
				rf.mu.Unlock()
			}
		case CANDIDATE:
			rf.mu.Lock()
			select {
			case <-rf.appendCh:
				rf.updateStateTo(FOLLOWER)
			case <-rf.electionTimer.C:
				rf.electionTimer.Reset(randElectionDuration())
				rf.startElection()
			default:
				if rf.voteAcquired > len(rf.peers)/2 {
					rf.updateStateTo(LEADER)
				}
			}
			rf.mu.Unlock()
		case LEADER:
			rf.broadcastAppendEnrties()
			time.Sleep(HEARTBEAT_INTERVAL * time.Millisecond)
		}

	}
```

### 初始化并开始循环
```go
func Make(peers []*labrpc.ClientEnd, me int,
	persister *Persister, applyCh chan ApplyMsg) *Raft {
	rf := &Raft{}
	rf.peers = peers
	rf.persister = persister
	rf.me = me

	// Your initialization code here (2A, 2B, 2C).
	rf.state = FOLLOWER
	rf.votedFor = -1
	rf.voteCh = make(chan struct{})
	rf.appendCh = make(chan struct{})

	// initialize from state persisted before a crash
	rf.readPersist(persister.ReadRaftState())

	go rf.startLoop()

	return rf
}
```

## 总结

---
第一次将论文中的算法“实现”，虽然是参考了大神的代码，还是觉得收获满满。
首先，纠正了我看论文一目十行的方法，以前总觉得论文这东西太虚，只是所谓的想法，经过这次无数遍的研读之后，发现论文中的每一句话对于整个算法的执行都有至关重要的作用。因此，以后在看论文的时候，需要逐句理解，最好能够抽象出整个模型，并且思考论文这么做对于整个系统的实现有什么好处
其次，自己的代码的能力实在是弱的不行，我能看得懂这些代码，并能在脑中把每一个函数链接起来，但是距离完全靠自己的能力去实现一个这样的系统还有相当长的路要走。
希望能坚持将自己所看的论文、方法都能用自己的话去说出来，加深下理解。







