# Zookeeper

## Feature
- file system abstraction
- Single Master(leader), multiple follower
- replicated databases(primary is at leader server), read are local, writes are forwarded to leader 
- quorum based commit
- atomic operations (Writes are either success or fail and no partial changes)
- Linearizable writes(central sequencer), reads may be stale.
  > Thus **highly scalable read** by adding more folower nodes. But **high write rates are bottleneck** to the system

## Feature Mechanism in ZK
- ZK provides `Watcher`, when the znode(a file/directory in zk abstraction) changes, server will send to those who are currently watching that data
- `Ephemeral znode`: temporary znode, lifetime is a session from client (a session starts when a client connects to zk). Upon disconnection, ephemeral znode will be deleted automatically
  > can use this feature to accomplish state **monitoring**, when a node went offline with crash, it can be easily detected by other nodes if other nodes current put a `watch` on a shared znode

## Practices

### Implementing a Load Balancer(Membership)
- Load Balancer software essentially is a software that tracks the current membership and some metadata(current load e.g) of server nodes

#### Procedure:
- each server node runs a ZK session (so the server node runs a server process that is a client connecting to ZK service), and creates a `ephemeral znode` under a aggreed location e.g., `/servers`. each created znode contains data about the servers ip and port number
- each client node also runs a ZK session(so the client device runs a client process that is a client connecting to ZK service). client puts a `watch` under `/servers`, and when client first ran, it will consult to ZK to see and maintain **a list of availible servers**.And it can then issue http request using this list. And if any server went down, client process will be notified and will update its list of availible servers.
  > **This client node is essentially a software level load balancer**. When outside request came in, it first will be directed to this process, and this process then direct the call to a designated server(it can be implemented so that it does some metadata check on the current traffic of each of individual servers that are alive)




## ZAB(Zookeeper Atomic Broadcast)
> **Important Assumption**: All writes goes into a **single leader server(central sequencer)** to assign an unique and monotonic increasing id. leader then replicate writes to follower servers

ZK就是分布式⼀致性问题的⼯业解决⽅案，**paxos**是其底层理论算法(晦涩难懂著名)，其中**zab**，**raft**和众多开源算法是对**paxos**的⼯业级实现。ZK没有完全采⽤**paxos**算法，⽽是使⽤了⼀种称为**Zookeeper Atomic Broadcast**（ZAB，Zookeeper原⼦消息⼴播协议）的协议作为其数据⼀致性的核⼼算法.

### ZAB Implementation
- similar to two-phase commit, where all writes goes to single leader, and leader assigns a unique id and commits(replicates) updates to followers. It is guaranteed that only commited transactions will be visible
### ZAB guarantees
- **原子广播协议 (Linearizable Write)**
  - Leader broadcast its updates to all followers with quorom aggrements
  - **monotonic increasing id** ensures the order of operation will be finalized between all replicated datastore. and `FIFO guarantee`(refer to mit zookeeper notes, or the zk paper) in Zookeeper allows the writes are executed in the order requested by client. **FIFO guarantee and global monotonic id ensures the global finalization ordering specified by the program**
- **崩溃恢复 (Leader Crash Recovery)**
  - upon leader crash. pick from all followers with the **biggest `txid`**. Since `txid` appeared in followers are **already commited transaction** (and a commited transaction means more than quorom # of nodes aggreed on that transaction), the follower with biggest `txid` is thus having the most recent commit transaction and is an eligible leader candidate

