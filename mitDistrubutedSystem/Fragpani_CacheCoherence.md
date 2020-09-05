# Define Cache Coherence

在一个处理器系统中，主设备(CPU或者外部设备)进行存储器访问时，将试图从存储器系统(主存储器或者其他CPU的Cache)中获得最新的数据拷贝。如果该主设备访问的数据没有在本地命中时，将从其他CPU的Cache中获取数据，**如果这些数据仍然没有在其他CPU的Cache中命中**，主存储器将提供数据。外设设备进行存储器访问时，也需要进行Cache共享一致性。

## Fragpani
- Distributed file system coordinated by distrubuted lock service
- Components
  - Fragpani client: just a normal client software providing normal interfaces for clients to operate the filesystem; Filesystem is virtualized to be local, where in reality Fragpani client communicate with remote Fragpani server to acquire locks and write/read data
  - Fragpani server: tracks the metadata about the filesystem
  - Fragpani lock service server: on the same node running the Fragpani server, grants and tracks the current owner of the locks
  - Fragpani shared disk storage: disk storage is partitioned and each partition has its own `Fragpani server` and F`ragpani lock service server`
## Fragpani Solution to Cache Coherence: Multi-reader/Single Writer Lock
- in Fragpani, each client has a Fragpani client, and once a file is read, it will be cached at local client, and all following updates by the client will only be written in the local memory, thus improves the write performance significantly
  > since this architecture means other clients read the same file might get **stale** copy of the data. Fragpani's solution to this is **Multi-reader/Single Writer Lock**
  - each client trying to read/write a file must acquire a lock from the Fragpani distributed lock service(those LS runs on the same nodes as the `Fragani server` process); 
  - multiple clients can acquire reader lock as long as no writer lock associated to file x is granted yet. Only a single writer lock can be granted at all tune associated to x; **To improve lock communication accross LS servers**, the locks are released in a **lazy manner**, meaning once a `client a` currently is holding a `lock L` associating with `file x` and is not currently working on x, it will still hold the lock. Util otherwise a `client b` request from LS server for lock L inorder to access x, LS will ask `a` to release lock. Such lazy manner improves the performance of the system due to reduced lock communication accross LS servers
  - `sync` in Fragpani **ensures the fault tolerance property**. since must recent writes only resides on the cache(memory) of the node currently holding the data. `sync` is ran every 30 seconds like in UNIX system so every once a while all dirty writes are flushed into disk
- Usage of locks essentially Fragpani to provide cache coherence since any client can read the most recent copy of data of acquired the lock; Fragpani also uses locks to provide atomic trasactions where all locks associated with all resources need to be acquire in order to execute the trasaction, and during the transaction since no other client can get the lock, we ensured the trasaction is atomic and not impacted by other concurrent operations
- Fragpani also uses **WAL** for updates on the filesystem(inodes, directories, new files created/deleted); WAL enabled crash recovery for Fragpani
  > Comparing to most other system implementaion of **WAL** where log file resides local to the machine, Fragpani client stores the log both in local disk and in remote filesystem(so each client has its own block on the remote shared disk storing the log for taht client); Upon releasing a lock(since that's when client has dirty writes, the Fragpani client first write the log to the remote disk and then write the modified data, and then release the lock)
  >
  > each log line is asscociated with a **version number**. so when replaying the log, we need to ensure the replay is not outdated. (consider the sequence of operation below)

  |client|time1|time2|time3|time4|result|
  |---|---|---|---|---|---|
  |A  | delete x|crash | | |BOOM! x is deleted!|
  |B  |         |      |create x||BOOM!|
  |C  |         |      |        | replay A's log(delete x)|BOOM!|

- Above we've been saying writing/reading remote disk, but all writing/reading to remote disk are provided by a driver software called `Petal`, so we write/read through `Petal` instead of directly accesing the shared disk