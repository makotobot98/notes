# Causal Consistency
- goal is to achieve **local write** and **local read** without iccuring the anomolies imposed by eventual consistency. 
  > **local write and local read**: when there's multiple data center DC1, DC2, DC3, each DC is its own a cluster of nodes and records are partitioned. client writes/reads to DC1, when client writes to DC1 without waiting for DC1 replicates to DC2 and DC3, it's called 'local write'. Similar to 'local read'
  >
  > **Eventual Consistency Anomolies**: when write m1 and m2 to DC1, and let DC1 ascynchronously replicate to DC2 and DC3, it might be the case that another client read DC2 m2 without seeing the m1 yet cause m1 is still not arrived to DC2. (Alice did 2 put operation `PUT1(upLoadPhoto)`, `PUT2(upLoadGallery)` where Photo is linked to gallery display to DC1, Bob did read operation on DC2 on `READ(Gallery)`, it might be the case that Bob saw the Gallery and the link to the photo, but the photo is not there yet)

## strawman solution 1: use timestamp to order events;
However physical clock synchronization between DCs is not accurate and there are problems of clock drifts, which makes physical clock synchronization difficult(possible but expensive)
  - Use Logical Clock: use **Lamport Timestamp**, where each timestamp is of `<monotonic_increasing_integer, DCid>`, Lamport Timestamp allows for total order of events. However, preserving the total order cannot help us to resolve the anomolies mentioned above. There must be an **aggrement of  finalization** of ordering between all nodes. This means we need to preserve the **causality** accross all nodes(so when Bob see the Gallery, we know Gallery display is **causally dependent** on Alice photo uploads, so we must make sure photo is already there before display gallery info to Bob)

## strawman solution 2: application level preservation of causal consistency; 

This needs the programmer to be careful about what write operation might cause the following read to be causally dependent, and thus enforce a sync operation/barrier for such writes. (in the example, programmer needs to sync when Alice do the `PUT1(uploadPhoto)` to both DC2 and DC3 before execute `PUT2(uploadGallery)`); **the sync here essentially block the DC1 until it makes sure `PUT1(uploadPhoto)` is propogated to DC2 and DC3**, and then its safe to execute `PUT2(uploadGallery)`
  > this is similar basic framework AWS Dynamo and Cassandra is based on. where for fault tolerance the sync operation uses **quorums** for majority of other replicas to sync instead of all. 
  >
  > in the solution, **read** is still local and performance is still good, but writes will be slow
## strawman 3: log based replication;
Each DC has a **centralized log server**, that does the AOF job on each client WRITE, and asynchronously send the log to all other DC to keep the order of WRITES consistent accross all nodes. This way we dont need to let the client sync and block until the writes are received by other DC, since in LOG file the order is garanteed. The **disadvantage** of this approach is scalibility, when there are many concurrent clients which are imposing high concurrent write workloads on the log server, the **log server will experience significant performance degration**

## COPS solution: causality dependency graph; 
> reference: [Scalable Causal Consistency for Wide-Area Storage with COPS](https://www.cs.cmu.edu/~dga/papers/cops-sosp2011.pdf)

What COPS does is similar to log based replication, instead of keeping a centralized log server, COPS maps each client to a `ClientContext`, where the `ClientContext` tracks the **version number**(monotonically increasing thus version numbers tracks the causality of a single client) and **dependency** for that operation.
  > for transactions, its necessary to preserver the entire dependency graph for each operation; whereas for a single operation, we simply need to track the previous dependent operation, since this dependency is one way

So instead of sending the entire log, COPS DS1 only needs to send updates(along with the version number and dependency) to DS2 and DS3.
  > At Bob's client on DS2, when it sees the `<PUT2(uploadGallery), version=2, dep=1>`, it will wait until the dependent write(photo WRITE opeartion) received before revealing to the client
  >
  > In general, if say DS1 sending WRITE operation to DS2 and DS3 in the order of `<xWrite3, v1, dep=NULL>`, `<yWrite5, v2, dep=v1>`, `<zWrite6, v3, dep=v2>`. and if DS2 only receives `<zWrite6, v3, dep=v2>` and client at DS2 is requesting to `read(z)`, the client knows the dependent updates have not yet seen and might be causally dependent, thus will block until opeartions corresponds to `v1` and `v2` arrive

- COPS approach in causal consistency eventually achieved **local write** and **local read**, especially for writes are asynchronous and it's garanteed the order of updates in replicas are aggreed accross the cluster
  - Comparing to strawman 2(block until replicated): COPS solution does not block the client for potential future causally dependent operation
  - Comparing to strawman 3(log based replication): COPS achieve a good performance where no longer need a central log server, and updates are streamed while causal consistency is preserved
- Another advantage of COPS implementation of causal consistency is that for writes that are **concurrent**(operation a and b are concurrent of a is not causally dependent on b and b is not causally dependent on a), the **system is not obligated to preserve their order and thus improve the efficiency while preserving the correctness of the system**
- There is an optimization in COPS non-transactional system(COPS has for transactional system), where its only necessary to preserve the previous dependent operation(instead of the entire dependency graph), due to the dependency is one way when it's non-transaction
- **There is still a big performance trade off in comparing to many eventual consistent systems, due to many information will be tracked and pushed around for tracking the causality**
- **COPS** can only support for one client connection in order to track all the `ClientConext` information. And this is the problem of COPS, since in real word, user might be connect to the different webservers, using different devices, it became hard to maintain one context per user
- One critisizm on critical consistency of COPS approach is the latency, which the dependency waits maybe **cascading**, and can be eventually very long delay for the client.

# Define Causality
> **The Story**: we have three data center (DC's), Alice is on DC1, and Bob is on DC2; Alice did 2 put operation `PUT1(upLoadPhoto)`, `PUT2(upLoadGallery)` where Photo is linked to gallery display to DC1, Bob did read operation on DC2 on `READ(Gallery)`, it might be the case that Bob saw the Gallery and the link to the photo, but the photo is not there yet

## **Three rules define potential causality, denoted by `->`**

1. **Execution Thread**. If a and b are two operations in a single thread of execution, then a -> b if operation a happens before operation b.
  > so Alices two operations a = `PUT1(upLoadPhoto)`, b = `PUT2(upLoadGallery)`, a and b are causally dependent since they are from one client process
2. **Gets From**. If a is a put operation and b is a get operation that returns the value written by a, then a -> b.
  > in the case of BOB, since BOB's `READ(Gallery)` := link to alice photo, we say BOB's `READ(Gallery)` is causally dependent on alice's `PUT2(upLoadGallery)`
3. **Transitivity**. For operations a, b, and c, if a -> b and b -> c, then a -> c.
  > Bob's `READ(Gallery)` -> Alice's `PUT2(upLoadGallery)`
  >
  > and Alice's `PUT2(upLoadGallery)` -> Alice's `PUT1(upLoadPhoto)`;
  >
  > we then say Bob's `READ(Gallery)` -> Alice's `PUT1(upLoadPhoto)`;
