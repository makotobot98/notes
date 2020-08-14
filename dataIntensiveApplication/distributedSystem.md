# problems with distributed system
- clock synchronicity
- process pause (GC collection), which stops in execution for a substancial amount of time, and once treated dead by other nodes, the process comes back without realizing its been treated dead. (split brain e.g.)
- asynchronicity and unreliability of the network
  - a packet loss does not tell if its network delay or node failure
  - even though protocols like TCP exists, it nevertheless introduces latency with flow control, packet queueing, and machenisms to ensure reliable data transfer
- Byzantine problem: where a node might send misleading messages to deceive other nodes

# Consensus
**consensus** is getting all of the nodes to aggree on something


## Trasaction isolation vs Distributed consistency
transaction isolation is primarily about **avoiding race conditions** due to concurrently executing transactions, whereas distributed consitency is mostly about **coordinating the state of replicas in the face of delays and faults**

## Linearizability
- basic idea of Linearizability is to make a system appear as if they were only one copy of the data, and all operations on it are atomic. With this guarantee, even though there may be multiple replicas in reality, the application does not need to worry about them.
- Distinguish **Serializability** and **Linearizability**, pg329
  - Serializability is an **isolation property of transactions**, where each transaction may read and write single/multiple objects(rows, documents, records). It guarantees that transactions behave the same as if they had executed in some serial order (each transaction running to completion before the next transaction starts). It is okay for that serial order to be different order in which transactions were actually run
  - Linearizability is a recency guarantee on reads and writes of a **register**(a register can be a row, object, a JSON document). **It does't group operations together into transactions, so it does not prevent problems such as write skew**, unless you take additional measures such as materializing conflicts
- Use case of Linearizability
  - after split brain in a single leader, usually the rest of nodes will elect a new leader by passing a lock, who succeeds in acquiring the lock becomes the leader. no matter how this lock is implemented, it must be linearizable where **all nodes must agree which node owns the lock**

## CAP Theorem
CAP sometimes means to pick 2 out of 3 from **Consistency(Linearizability), Availibility, Partition Tolerance**, but this is misleading. **Network Partitions** are a kind of fault, so they aren't something about which you have a choice, they will happen no matter what.
> At times when the network is working correctly, a system can provide both consistency(linearizability) and total availibility. **When a network fault occurs, you have to choose between either linearizability or total availibility** Thus, a better way of phrasing CAP would be **either Consistent or Availible** when Partitioned. 


## Causally Consistent
two operation r1 and r2 are **causally dependent** if r2's input is related to the r1's output or vise versa. two operation r1 and r2 are **concurrent** if r1 and r2 are not related and results of r1 and r2 are independent from each other.

### Lamport timestamp
- How does it work: Lamport timestamp is a pair of `(counter, nodeID)` every node and every client keeps track of the maximum counter value it has seen so far, and includes that maximum on every request. When a node receives a request or response with a maximum counter value greater than its own counter value, it immediately increases its own counter to that maximum
- Lamport timestamp provide a total ordering consistent with causality. It does not tell whether two operations are concurrent or whether they are causally dependent, but **enforces the total ordering of timestamp** (this timestamp is independent from a physical time-of-day clock)

### Total order broadcast
- Lamport timestamp is sufficient to provide a total order within a single node, for the consensus to work accross a cluster, we need **majority** of the nodes agree on the **order** of operations each node had executed
- **Total order broadcast** is ussaly described as a protocol for exchanging messages between nodes and it guarantees:
  - **Reliable delivery**: no messages are lost, meaning mesasges are delivered to non-faulty nodes
  - **Total ordered delivery**: messages are delivered to every node in the same order
- to put it another way, imagine delivering a message is like appending to the log, since all nodes must deliver the same msaages in the same order, all nodes can read the log and see the same sequence of messages
- **Total order broadcast vs Linearizability**: they are different.
  - total order broadcast is asynchronous: messages are guaranteed to be delivered reliably in a fixed order, but there is **no guarantee about when** a message will be delivered(so one recipient may lag behind the others).
  - Linearizability is a **recency guarantee**: a read is guaranteed to see the latest value written
#### Use cases
- total order broadcast can be used to implement 
  **serializable trasactions**: if every message represents a deterministic transaction to be executed as a stored procedure, and if every node processes those message **in the same order**, then the partitions and replicas of the database must be kept consistent with each other, thus a serializable transaction is guaranteed.
- Total order broadcast is also useful for implementing a lock service that provides fencing tokens, where each request to acquire the lock is appended as a message to the log, and all messages are sequentially numbered in the order they appear in the log
- Linearizable storage can be implemented using total order broadcast(pg350); total order broadcast can be implemented using linearizable storage(pg351)
  > this essentially means a linearizable compare-and-set register(accross the cluster) and total order broadcast are both equivalent, and equivalent to **consensus**. Meaning if we can solve one of those, we can transform it into the solution for the others







