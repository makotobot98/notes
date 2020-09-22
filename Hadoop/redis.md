# Intro
## Usage
- distributed cache
  - issues need to consider when using distributed cache service: 缓存雪崩，缓存击穿，缓存穿透
  - cache coherence: (use cache invalidation in look-aside cache FaceBook)
  - cache concurrency
- distributed lock service
- optimistic lock (object/row/db level compare and set)
- session seperation (session in server, so redis can serve as a session for the application server)
- message queue

## General Distribute cache read/write mode(缓存读写模式)
### Cache aside pattern (Look aside cache)
![](rsrc/redis_cache_aside.png)
#### pros

#### cons
- **possible dirty writes** (consider a read request after updating DB but not data in cache has not been deleted, read will resolve into a stale value)

### Read/Write through pattern
- read-through cache: when client performs read on the cache and cache does not have data, cache will first fetch the data from database then update its own copy
- write-through: updates are performed directly on the cache, afterwards cache will write to the database
  - this pattern causes the cache to be coupled with the application logic, harder for software evolution
### Write Behind Cache Pattern
- client update the cache on write, and cache will **asynchronously** (and perhaps in batch) update the database
- problem with this pattern is that possible data loss and data consistency, but highly performant


# Data Types
- key can only be strings
- value can be various types supported by Redis
## List
- can be used as a deque
- can also be used to implement message queue, since it allows to be blocked until elements in the list are availible
## Set
## SortedSet
- 有序集合： 元素本身是无序不重复的, 每个元素关联一个分数(score), 可按分数排序，分数可重复
## HashMap
## BitMap (HashSet)
- more memory efficient

# Data Persistence
## RDB
- periodically main redis process will fork a background process to copy the current data in a snapshot fashion

## AOF
- append only the **instruction**, not the data, for performance and fast recovery
![](rsrc/redis_aof_write_save.png)

- Following picture shows the state of the main redis process when performing aof flow:
  - `Write` always block the main process to write the update instruction to memory
  - `Save` is when persisting to disk. when mode is in `AOF_FSYNC_EVERYSEC`, main process will fork a child process to do the save work (note the `fork()` is blocking but really short amount of time needed)
  ![](rsrc/redis_aof_mode_compare.png)

- there will be a daemon process forked by main process to periodically compact the aof file

## AOF vs RDB
1. RDB采用二进制压缩储存，AOF存操作命令，采用文本储存。(如果用混合模式(RDB+AOF)则事文本+二进制压缩，也就是RDB已经做好的部分为二进制，而RDB后的新update instruction为文本)
2. RDB has good performance but potential data loss(from last snapshot); AOF is more real time but costly
  > but when data amount is too large, fork a process to do snapshot will be extremely expensive(recall forking is duplicating the parent process); thus when data amount(in a short period of time) is too large, it is recommanded to use AOF
3. it's sometimes to avoid using both AOF and RDB, reason is that it optimizes the performance of the Redis server; This approach should only be used if the original source the data are preserved somewhere else; using AOF and RDB will have some performance cost due to the restart/replay data/forking&blocking. 


# Transaction in Redis
- weak guarantee: if transaction fails, then partial write will occur if some update operations are successful (so does not provide Atomocity guarantee)
# Lua
- Redis uses the same Lua interpreter to run all the commands. Also **Redis guarantees that a script is executed in an atomic way: no other script or Redis command will be executed while a script is being executed. This semantic is similar to the one of MULTI / EXEC**. From the point of view of all the other clients the effects of a script are either still not visible or already completed.
- However this also means that executing slow scripts is not a good idea. It is not hard to create fast scripts, as the script overhead is very low, but if you are going to use slow scripts you should be aware that while the script is running no other client can execute commands.
  > note if using `MULTI/EXEC` transactions in Redis, other clients can still be runnning commands while one server is `EXEC` transactions
## Comparing other ways to send commands in bulk
[Blog](https://engineering.linecorp.com/en/blog/redis-lua-scripting-atomic-processing-cache/)

- **pipeline**: the purpose of using pipeline is solely to optimize network bandwidth usage. Using pipeline **does not guarantee atomic operation**. To process commands atomically, you either have to use transactions or Lua script.
- **Redis Transaction**: atomicity is not strongly guaranteed, concurrent running transactions might affect one and another. **Note both Lua and Multi/Exec does not provide failure rollback, so partial results still occur**
  > 原文：“同 redis 的事务一样，那些通过 redis.call 函数已经执行过的指令对服务器状态产生影响是无法撤销的”
  >
  > 详细看rsrc/redisLua_原理.pdf
- **Lua**

# High Availibility
## Master-Slave replication(not cluster)
- master-slave architecture(and slaves can be master of other slaves); master accepts write/read, slave accept reads only
- synchronization
  - upon new slave starts, 先全量同步(copy snapshot)后增量同步(master sends all new writes to slave)
- slave sends heart beat to master periodically

## [Sentinel](https://redis.io/topics/sentinel)
- a cluster of sentinel nodes will elect new leader upon leader failure. Sentinel cluster will track the health of the cluster
- [CNBlog](https://www.cnblogs.com/williamjie/p/9505782.html)
### Mechanism
- communication between sentinel nodes: using `publish-subscribe` in Redis, each sentinel publish to channel `sentinel:hello` every 2s informing its port,ip,...etc metadata. 
- failure detection: sentinel sends pings to redis nodes periodically, and if no reply the node will be considered 'down', and if quorom # of sentinel nodes aggree on the same node is down, redis cluster will perform leader/node failover
  > Leader Election: if no sentinel cluster used, Raft is used as Leader Election; if sentinel cluster is used, sentinel cluster will using quorom to decide the new leader

## Consistent Hashing
- hash partitioning is efficient and allowing balanced distribution, however when it comes to rebalancing it became costly to recompute all the hashes. `Consistent Hashing` is used to reduce the amount of data that needs to be recomputing hash code
- 虚拟槽分区: 虚拟槽分区是Redis Cluster采用的分区方式, 预设虚拟槽，每个槽就相当于一个数字，有一定范围。每个槽映射一个数据子集，一般比节点数大
  > 虚拟槽是基于consistnet hashing的分区方式，解决了consistent hashing选区必须选均匀的分区点的问题，因为虚拟槽可以map到任意一个物理的分区点，所以不用担心如何均匀分布物理分区

## Redis Cluster
![](rsrc/redis_cluster_arch.png)
- Redis Cluster refers to a cluster of cluster of master-slave nodes. Within this cluster data is partitioned accross different master nodes, within the small cluster (individual single master-slaves cluster), master is responsible for write/read, slaves are read only.
- RedisCluster是由多个Redis节点组构成，是一个P2P无中心节点的集群架构，通过Gossip协议传播信息; 通过Gossip协议，cluster金额图提供集群时间状态同步更新，选举自助failover等功能
- 为什么需要Redis Cluster(Comparing to single master-slave replication):
  1. 主从复制不能实现高可用
  2. 随着公司发展，用户数量增多，并发越来越多，业务需要更高的QPS，而主从复制中**单机**的QPS可能无法满足业务需求
  3. 数据量的考虑，现有服务器内存不能满足业务数据的需要时，单纯向服务器添加内存不能达到要求，此时需要考虑分布式需求，把数据分布到不同服务器上
  4. 网络流量需求：业务的流量已经超过服务器的网卡的上限值，可以考虑使用分布式来进行分流
- 客户端访问任意节点时，对数据的key按照CRC16规则进行hash运算取余，**如果余数在当前访问的节点管理的槽范围内，则直接返回对应的数据如果不在当前节点负责管理的槽范围内，则会告诉客户端去哪个节点获取数据，由客户端去正确的节点获取数据**(这个过程叫做`move redirection`)
  > (槽(slots)平均分配给节点进行管理，每个节点只能对自己负责的槽进行读写操作由于每个节点之间都彼此通信，每个节点都知道另外节点负责管理的槽范围。在集群刚启动时，槽会平均并连续地分布到每个master节点)
- [CNBlog](https://www.cnblogs.com/williamjie/p/11132211.html)

### Failure Detection
`Redis Cluster`通过`ping/pong`消息实现故障发现：不需要`sentinel`

`ping/pong`不仅能传递节点与槽的对应消息，也能传递其他状态，比如：节点主从状态，节点故障等

故障发现就是通过这种模式来实现，分为`主观下线`和`客观下线`
- 主观下线：某个节点认为另一个节点不可用，'偏见'，只代表一个节点对另一个节点的判断，不代表所有节点的认知
- 客观下线：当半数以上持有槽的主节点都标记某节点主观下线时，可以保证判断的公平性
### Failover
对从节点的资格进行检查，只有难过检查的从节点才可以开始进行故障恢复, 使偏移量最大的从节点具备优先级成为主节点的条件
![](rsrc/redis_cluster_failover_vote.png)
![](rsrc/redis_cluster_failover_election.png)
### Advantage
- 高性能
  - 多个主节点，负载均衡，读写分离
- 高可用
  - faliover
- 易扩展
  - 向redis cluster添加/溢出节点很容易，不需要停机
  - 数据分区，海量存储
### Disadvantages
- 当节点数量很多时，性能不会很高
  > 解决方式：使用**智能客户端**。智能客户端知道由哪个节点负责管理哪个槽，而且当节点与槽的映射关系发生改变时，客户端也会知道这个改变，这是一种非常高效的方式

## Smart智能客户端
- Smart智能客户端用途在于客户端自己保留slots->node的mapping，(这样也就不会在查某个key时询问连接的redis node当前查询的key的数据在哪个槽)
- `JedisCluster`是Jedis根据RedisCluster的特性提供的集群只能客户端; `JedisCluster`为每个节点创建连接池，并和各个节点建立映射关系缓存(cluster slots)，`JedisCluster`为每个主节点负责的槽位一一与主节点连接池建立映射缓存；`JedisCluster`启动时，已经知道keys,slots和node之间的关系，可以直接找到目标节点
# Tips

## Reaching maxmemory
- upon reaching memory, redis will have to request for more virtual memory pages, which will cause page swaps and potential threshing while performance dramatically fall
- solution to this is to set timeout for records, and set threshold for `maxmemory` allowed for each redis node (e.g., 3/4 physical memory if a node is fully dedicated as a redis server)
  - upon reaching `maxmemory`, records will be cleaned based on LRU

## 缓存穿透
缓存穿透是指在高并发下查询不存在的key，会穿过缓存查询数据库。导致数据库压力过大而宕机

缓存穿透是指缓存和数据库中都没有的数据，而用户不断发起请求，我们数据库的 id 都是1开始自增上去的，如发起为id值为 -1 的数据或 id 为特别大不存在的数据。这时的用户很可能是攻击者，攻击会导致数据库压力过大，严重会击垮数据库。
> 像这种你如果不对参数做校验，数据库id都是大于0的，我一直用小于0的参数去请求你，每次都能绕开Redis直接打到数据库，数据库也查不到，每次都这样，并发高点就容易崩掉了。
- 解决方案1：在接口层增加校验，比如用户鉴权校验，参数做校验，不合法的参数直接代码Return，比如：id 做基础校验，id <=0的直接拦截等。从缓存取不到的数据，在数据库中也没有取到，**这时也可以将对应Key的Value对写为null、位置错误、稍后重试这样的值具体取啥问产品，或者看具体的场景，缓存有效时间可以设置短点，如30秒（设置太长会导致正常情况也没法使用）。** 这样可以防止攻击用户反复用同一个id暴力攻击，但是我们要知道正常用户是不会在单秒内发起这么多次请求的，那网关层Nginx本渣我也记得有配置项，可以让运维大大对单个IP每秒访问次数超出阈值的IP都拉黑。
- 解决方案2(Bloom Filter)：利用高效的数据结构和算法快速判断出你这个**Key是否在数据库中存在**，不存在你return就好了，存在你就去查了DB刷新KV再return
  > Bloom Filter: Suppose we have a set HashFunc = {f1, f2, f3 ...}, for each incoming key = x, we compute and set d1 = f1(x) % n, d2 = f1(x) % n, ...
   where n = # of slots we have. we set d1-th, d2-th, d3-th ... positions to 1 in a bit string of length n. this bit string will be used as our bloom filter.
  > 
  > Properties of Bloom Filter: given a incoming query key y and a set of hash functions {f1, f2, f3 ...}, we compute di = fi(y) % n and check if all di-th bits in the `bloom filter` bit string are set to 1
  > - if all are set to 1, it is possible for y to be hashed before; 
  > - if not all bits are set to 1, it is guaranteed that y has not been hashed before; (and thus does not exist)
  > - more hash functions we have, higher the accuracy of bloom filter, but it may be computationally more expensive; in the same time, # of space used by bloom filter is extremely efficient
  - **Integrating Bloom Filter**: we have a bloom filter bit string at cache layer, and when qeury comes in, and the cache does not have the corresponding data. We first check the blook filter before querying the database, if bloom filter is positive about data is not in the database, we simply return to the client
## 缓存雪崩
缓存雪崩是指在我们设置缓存时采用了相同的过期时间，导致缓存在某一时刻同时失效，请求全部转发到DB，DB瞬时压力过重雪崩。


> 举个简单的例子：如果所有首页的Key失效时间都是12小时，中午12点刷新的，我零点有个秒杀活动大量用户涌入，假设当时每秒 6000 个请求，本来缓存在可以扛住每秒 5000 个请求，但是缓存当时所有的Key都失效了。此时 1 秒 6000 个请求全部落数据库，数据库必然扛不住，它会报一下警，真实情况可能DBA都没反应过来就直接挂了。此时，如果没用什么特别的方案来处理这个故障，DBA 很着急，重启数据库，但是数据库立马又被新的流量给打死了。这就是我理解的缓存雪崩。
>
> **同一时间大面积失效，那一瞬间Redis跟没有一样，那这个数量级别的请求直接打到数据库几乎是灾难性的，你想想如果打挂的是一个用户服务的库，那其他依赖他的库所有的接口几乎都会报错，如果没做熔断等策略基本上就是瞬间挂一片的节奏，你怎么重启用户都会把你打挂，等你能重启的时候，用户早就睡觉去了，并且对你的产品失去了信心，什么垃圾产品。**

- 解决方案: 处理缓存雪崩简单，在批量往Redis存数据的时候，把每个Key的失效时间都加个随机值就好了，这样可以保证数据不会在同一时间大面积失效，我相信，Redis这点流量还是顶得住的。
```
setRedis(Key，value，time + Math.random() * 10000);
```

## 缓存击穿
缓存雪崩是因为大面积的缓存失效，打崩了DB，而缓存击穿不同的是缓存击穿是指一个**Key非常热点**，在不停的扛着大并发，大并发集中对这一个点进行访问，当这个Key在失效的瞬间，持续的大并发就穿破缓存，直接请求数据库，就像在一个完好无损的桶上凿开了一个洞

- 解决方案1： 分布式锁- 所有客户端进来的读请求都需获取锁，这样在热点时效的时候只有第一个客户端进来会cache miss并且从数据库读到并更新到缓存新的数据，这样后来的客户端的读请求就不会出现cache miss也就不会出现缓存穿透；由于使用了锁，应用程序的性能会下降很多
  > a variation of this scheme will be: when there is a cache miss, the first client on the cache miss will obtain the lock, and use the lock to query the database (**FaceBook Memcache**), once the data is read the cache will be updated, the subsequent read request will not resolve cache miss
- 解决方案2： 热点key不设失效时间， 但是会造成写一致性问题

## cache coherence
- [分布式之数据库和缓存双写一致性方案解析（双删延时+异步消息）](rsrc/分布式之数据库和缓存双写一致性方案解析（双删延时+异步消息）.pdf)

解决方案：
1. set timeout in cache, so records will eventually stay consistent with whats in database; However, this causes dirty read a lot with stale data
2. 先删数据库->再删缓存/先删缓存->再删数据库：仍然有脏读问题，具体看pdf
3. 延时双删： 避免了大多数情况下的脏读，但是如果删除失败，还是会造成脏读
4. 异步消息：使用MQ来确保删除操作一定成功，但由于是异步，效率会很慢


# Q
- so Lua scripts cannot guarantee atomicity? meaning partial results can still occur upon failure/error that interrupt the execution
  > Yes, https://www.secpulse.com/archives/78350.html; Both Lua and Redis Transaction cannot guarantee to rollback the partial writes upon failure

- How does Redis guarantee only one server executes the Lua while others will not(refering `no other script or Redis command will be executed while a script is being executed`)? is there a synchonization mechanism deployed?
