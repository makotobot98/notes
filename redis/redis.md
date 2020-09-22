# Data Structures
- `LISTs` use minimal memory compared to some other structures
# [Data Persistence](https://redislabs.com/ebook/part-2-core-concepts/chapter-4-keeping-data-safe-and-ensuring-performance/4-1-persistence-options/)

## Snapshot(RDB)
- memory might be overwhelmed if data is too big(tens of gigabytes) since snapshot makes heavy uses of memory (which may cause heavy use of virtual memory, and degrade the performance)
- data may go out of sync due to the snapshot time configuration(if snapshot period is set too long)

## AOF
- real time appending to disk(where when to flush is determined by OS, recall the writeFileSync() means to block the process(redis) until the write on disk is successful), can usually set sync period be 1 sec. (**Do not set `appendfsync always` which write every change to disk as they happen** due to write amplification from continuously writing small amounts of data to the end of a file)
- more resilient to data loss if a few minutes of data loss is not acceptible
- downside: the append-only log file will continuously grow. Over time, a growing AOF could **cause your disk to run out of space**, but more commonly, upon restart, Redis will be executing every command in the AOF in order. When handling large AOFs, **Redis can take a very long time to start up**.
  - **Solution**: Rewriting/compacting append-only files feature supported by Redis

## [RDB vs AOF](https://redis.io/topics/persistence)
# Replication
- Replication process in a master-slave fashion: upon slave request for a `slaveof` to the master, master will `bgsave` demeaon process to take a snapshot while keeping a backlog of writes happened after the snapshot. and send those to slave
- COMMANDThe `INFO` command can offer a wide range of information about the current status of a Redis server—memory used, the number of connected clients, if the Redis server is synced with master Redis server, the number of keys in each database, the number of commands executed since the last snapshot, and more. Generally speaking, `INFO`  is a good source of information about the general state of our Redis servers, and many resources online can explain more. 

# Redis Sentinel
One recent addition to the list of Redis tools that can be used to help with replication and failover is known as Redis Sentinel. Redis Sentinel is a mode of the Redis server binary where it doesn’t act as a typical Redis server. Instead, Sentinel watches the behavior and health of a number of masters and their slaves. By using `PUBLISH/SUBSCRIBE` against the masters combined with `PING` calls to slaves and masters, a collection of Sentinel processes independently discover information about available slaves and other Sentinels. Upon master failure, a single Sentinel will be chosen based on information that all Sentinels have and will choose a new master server from the existing slaves. After that slave is turned into a master, the Sentinel will move the slaves over to the new master (by default one at a time, but this can be configured to a higher number).

# Scaling
## use tips:
- If we’re using small structures (as we discussed in chapter 9), first make sure
that our max ziplist size isn’t too large to cause performance penalties.
- Remember to use structures that offer good performance for the types of queries
we want to perform (don’t treat LISTs like SETs; don’t fetch an entire HASH
just to sort on the client—use a ZSET; and so on).
- If we’re sending large objects to Redis for caching, consider compressing the
data to reduce network bandwidth for reads and writes (compare lz4, gzip, and
bzip2 to determine which offers the best trade-offs for size/performance for
our uses).
- Remember to use pipelining (with or without transactions, depending on our
requirements) and connection pooling, as we discussed in chapter 4.

## Scale Reads
- add Redis read-only slaves servers
- encrypt/dycrypt(AES-128, RC4, ..etc) the data read to reduce the network bandwidth usage and overall efficiency

## Scale Writes
- partitioning accross more Redis nodes

# 缓存可能面临的问题
- 缓存雪崩: keys expiring around a time t, and at time t, there is an intensive query on the application, causing all cache miss and all queries went directly to the database causing database to be overloaded
- 缓存穿透: an intense pattern of queries on values exist in neither cache nor database, causing the database to crash
- 缓存击穿: an intense pattern of queries on hot keys, when hot keys expired, all those queries went directly into the database overloading database