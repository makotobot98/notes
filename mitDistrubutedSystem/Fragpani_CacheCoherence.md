# Define Cache Coherence

## Fragpani
- Distributed file system coordinated by lock
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