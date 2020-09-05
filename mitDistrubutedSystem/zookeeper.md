# Zookeeper
## Linearizability Definition
- A **history** is a sequence of operations(read and write) made on an object by a set of threads or processes; 
> A history is linearizable if: there exists a total order of operations matches **real time** and **aggreed by all processes/threads**. All read/writes see preceding read/write in the order

**More illustrating Linearizibility**
```
P1: |- Wx(0) -|   |- Wx(1) -|
P2:               |- Wx(2) -|
P3:          |- Rx(2) - | |- Rx(1) -|
---------------------------------------------
*Horizental axis is the point in time 

here P1's Wx(1), P2's Wx(2), and P3's Rx(2) and Rx(1) are all concurrent

The question is whether or not the above history is linearizable:

Yes, and a valid order is: 
P1's Wx(0) -> P2's Wx(2) -> P3's Rx(2) -> P1's Wx(1) -> P3's Rx(1)

```

### Comparing Linearizability with Sequential Consistency
- linearizability(or strong consistency) which maintains a global, real-time ordering;
- sequential consistency, which ensures at least a global ordering, which may NOT be **real-time**
  > **causal consistency**, which ensures **partial orderings** between **dependent operations**, so ordering between concurrent operations are not guaranteed

Consider the following sequence operations executed by P1 and P2, where W(x)a means to write to a data object x with value of a:

|process|t=1|t=2|t=3|t=4|
|---|---|---|---|---|
|P1 |W(x)a|||
|P2 |-|R(x)Nil|R(x)a||

Note that P2 and P1 both obeies a global ordering where P2 read x = `a` (**Not in real time**, bc we know by the time t = 2 when P2 read x, P1 finishes writing at t = 1, **which is before! not concurrent!**). However, under linearizability, **this is a vialation** to its consistency requirement in which at t = 2 `P2: R(x) = Nil` happens in real time before P1: `W(x)a`, so it should not return Nil but instead return a; So following would satisfy requirements for linearizibility:

|process|t=1|t=2|t=3|t=4|
|---|---|---|---|---|
|P1 |W(x)a|||
|P2 |-|**R(x)a**|R(x)a||

> **Sequential Consistency**: The result of any execution is the same as if the (read and write) operations by **all processes** on the data store were executed in some sequential order and the operations of each individual process appear in this sequence in the order specified by its program

## Guarantees in Zookeeper
- **Linearizable Write, Read may be stale**
   > This guarantee translates to the underlying architecture of ZK: All writes goes into a single server(the **sequencer** to add monotomic id), thus order is guaranteed since handled by central sequencer. Reads goes to the replicated database in any of the followers, which may produce the stale data
   >> Writes in leader server are performed in quorom to get aggrements from followers and then commit, so some follower out of the quorom might not get the most recent data
- **FIFO execution order** in the order of requests submitted from the client. This guarantee is combined with the functionality that ZK offers: all operations submitted from client can be **asynchronous**

### Implementation of guarantee
- Implementation wise, it's fairly straight forward to provide the Linearizible Write and FIFO execution order, since all writes goes in the central leader(sequencer), so each **writes are tagged with an unique monotonic increasing ID(zxid)**, allowing operations to go in an **log order** and it's no difficult to get all server to aggree on such order

## Common primitives can be implemented using ZK
- Group Membership
- Locks
- reader/writer lock
- Double Barrier

## Q
- All writes goes in a single Leader in ZK, scalability and performance? Leader is the bottleneck
> Yep, writes are bottleneck, but ZK works well with heavy read workloads, and read scales linearly with added machines
- If the case when client A reads follower server F1, and got `zxid = x1`, and based on this value perform `W(x)y` with version `x2`. However, the data in F1 is behind which in server the zxid for x is `x3` already. What happen now when the W(x)y with outdated `zxid = x2` sends to server? **This question is more concerning when different clients performs write follow read operations on different ZK server(this problem wont exist if clients are independent so they perform on different data object, and we have FIFO guarantee from ZK), which one write might based on a read that's stale**
> In this senario, it is disasterous. However, in ZK, `watch` mechanism does allows us to resolve situations like this. Consider the following Example where we implement Dynamic Configuration Management in Distributed System(in [zookeeper paper](https://pdos.csail.mit.edu/6.824/papers/zookeeper.pdf) 2.4):

```
set up: a new master node just came online and want to change configuration files for the cluster. By following the example in 2.4, we delete a ZK file znode 'ready', so everybody in the cluster where pause upon realize the 'ready' znode is not present, in the meantime master make the configuration changes(by writing to some aggreed location). When master is finished, it creates the 'ready' znode, and then the new configuration is availible for other nodes in the cluster. 
```

> A valid order of operations in the cluster looks like this where we consider single master and a single client in the cluster reading the configurations files `fi` (from top to bottom is the sequence of execution, same row means at same point in time)

|Master               |       Client             |
|         ---         |          ---             |
|`Delete('ready')`    |                          |
|`Write('f1')`        |                          |
|`Write('f2')`        |                          |
|`Create('ready')`    |                          |
|                     |`Exist('ready')->true`    |
|                     |`Read('f1')->updated f1`  |
|                     |`exist('f2')->updated f2` |

> A disasterous order will occur if client calls `Exist('ready')` before Master's `Delete('ready')`, and then while Master is writing f1, client will only read the partial and possibly wrong configurations in `f1` and `f2`

|Master               |       Client             |
|         ---         |          ---             |
|                     |`Exist('ready')->true`    |
|`Delete('ready')`    |                          |
|                     |`Read('f1')->old f1`      |
|`Write('f1')`        |                          |
|`Write('f2')`        |                          |
|`Create('ready')`    |                          |
|                     |`exist('f2')->updated f2` |

**\*Note Above operation does not violate the guarantees provided by ZK: Writes are still linearized and executed in the order of client request(two clients: master and client node)**

> To resolve this, we use `watch` provided by Zookeeper, where when we read/write a value, we can attach a `watch` flag on a dataobject(in ZK, called a znode) `x`, and when leader server receives the `Update/Write()` request and sees the `x` is outdated in the replicated server where clients is connected to, it will send a **notification** to tell the client that the value of `x` is outdated(tho not telling what the new updated value is, client needs to request another poll for that value);
>
> And it is guaranteed that the notification will arrive before client resolves to any read locally to a dataobject x that might be dependent on consecutive writes in master. So with the notification out diagram will look like this

|Master               |       Client             |
|         ---         |          ---             |
|                     |`Exist('ready', watch=true)->true`    |
|`Delete('ready')`    |                          |
|                     |`Notification from master` -> client can implement its own logic here to handle, notification is guaranteed to arrive before client sees f1|
|                     |`Read('f1')->aborted`      |
|`Write('f1')`        |                          |
|`Write('f2')`        |                          |
|`Create('ready')`    |                          |
|                     | \*abort by client        |

**\*Here notification is guaranteed to arrive before client resolves to `Read('f1')`**; Another possible senario is like following, where notification is guaranteed to arrive before client resolves to `Read('f2')`

|Master               |       Client             |
|         ---         |          ---             |
|                     |`Exist('ready', watch=true)->true`    |
|                     |`Read('f1')->old f1`      |
|`Delete('ready')`    |                          |
|                     |`Notification from master` -> client can implement its own logic here to handle, **notification is guaranteed to arrive before client sees f2**|
|`Write('f1')`        |                          |
|`Write('f2')`        |                          |
|`Create('ready')`    |                          |
|                     | \*abort by client        |


