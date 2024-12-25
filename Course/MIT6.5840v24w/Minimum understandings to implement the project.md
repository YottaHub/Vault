- Run test many times:

```shell
# https://gist.github.com/JJGO/e64c0e8aedb5d464b5f79d3b12197338
python3.11 ../../dstest.py 3A --iter 10 --race 
```

# Lab 3: Raft

## Part 3A: leader election

### States

In ***Raft***, servers have three different states. And in a **==[[Minimum understandings to implement the project#Election|election]]==**, **candidate** starts a new term (expect leading).

```go
type RaftState int
const (
	Follower RaftState = iota
	Candidate
	Leader
)
func (r RaftState) ToString() string {
	return [...]string{"Follower", "Candidate", "Leader"}[r]
}
func (rf *Raft) UpdateStateUnSafe(term int) {
	if term > rf.currTerm {
		rf.currTerm = term
		rf.state = Follower
		rf.voteFor = RaftInvalidServer
	}
}
```

![[raft fig4 server states.png]]

### Heartbeat

Raft uses a **heartbeat** mechanism to trigger leader election.

**Leader**s send periodic heartbeats (`AppendEntriesRPC`s that carry no log entries) to all **followers** in order to maintain their authority.

![[raft fig2 append entries.png]]

### Election

If a follower receives no communication over a period of time called the *election timeout*, it increments its current term and transitions to **candidate** state. Then it votes itself and issues `RequestVote` RPCs in parallel to each of the other servers in the cluster.

![[raft fig2 request vote.png]]

Three things might happen:

![[raft fig5 timeline.png]]

1. It wins the **majority** election. The candidate sets its state to **leader** and sends heart beat messages to all of the other servers.
	- Each server votes at most one candidate in a given term **FCFS**-ly.
2. Another server established itself as leader.
	- From an election `AppendEntries` RPC, claimed leader's term $\ge$ candidate's current term; legitimate and returns to **follower** states.
	- else, rejects and continues in **candidate** state.
3. No winner. Restart election by incrementing its term and initiating another round of `RequestVote` RPCs with a new **randomized** *election timeout*.

![[raft fig2 rules for server.png]]

## Part 3B: Log

### ==**Log replication**==

1. **Client** request: `command`
2. (a) **Leader** appends the `command` to `rf.log` as a [[Minimum understandings to implement the project#`LogEntry`|LogEntry]]
2. (b) **Leader** issues `AppendEntries` RPC to replicate the `LogEntry` into each follower's `rf.log` in parallel
3. If ==**[[Minimum understandings to implement the project#Safety|safely replicated]]**==, **leader** applies the entry and `lastApplied` in leader becomes new `rf.commitIdx`
4. Returns the result of the execution to **Client**

![[raft fig1 replicated state machine arch.png]]

### `LogEntry`

- A state machine command, `command interface{}`
- The term number when the entry was received by the **leader**, `term int`
- An integer index identifying its position in the log, `idx int` for `rf.log`

```go
type LogEntry struct {
	command interface{}
	term    int
	idx     int
}
```

![[raft fig6 log.png]]


### Commit

- A log entry is *==commit==* ted once it has been replicated on a **majority** of the servers

### Consistency

- `AppendEntries` RPC revisit, notes that to for simplicity and avoiding non-existent network congestion, we assume `len(entries) <= 1`.
	```go
	type AppendEntriesArgs struct {
		Term         int
		LeaderId     int
		PrevLogIdx   int
		PrevLogTerm  int
		entries      []LogEntry
		LeaderCommit int
	}
	type AppendEntriesReply struct {
		Term int
		Ok   bool
	}
	```
- Consistency check: `follower.log[idx].idx == PrevLogIdx && follower.log[idx].term == PrevLogTerm`, else reply false. **Leader** notices inconsistencies from replies and forces consistency by log matching.
	- `nextIndices` to send for each **follower** that starts as `leader.lastApplied + 1`
	- **Follower** rejects if check fails and delete any entry after it. `leader.nextIndices[follower] -= 1` and retries.

### Safety

- In ***Raft***, `leader.log` is guaranteed to be all *committed* and never deletes.
	- 

![[raft fig8 time sequence.png]]
