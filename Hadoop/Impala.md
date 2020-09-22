# Impala

## Comparing Impala and Hive
- Impala did not use **MapReduce**. MR is a good framework for parallel processng, and is good for batch processing but not for **interactive SQL execution**.
  > Why MR is slow：1. for fault tolerance, there are a lot of IO during the shuffling, as well as between jobs(we call persisting the intermeidate results); 2. during shuffling records will be forcibly sorted by key
- Impala converts the query into a **Execution Tree** instead of a series of MR tasks. Intermediate results are polled in terms of streams, so less Disk IO operation;
- Impala tends to **utilize memory** more in comparing to disk, trades performance for fault tolerance
- no **cold start** issue in Impala comparing to Hive(recall each HQL query is a series of MR jobs, meaning to start new Mappers and Reducers for each MR job, this means frequent request/allocate resources(containers) from resource manager(recall `Yarn` architecture)). Impala utilizes local query(so no costs for allocating resources), and there is a continuously running Impala daemon when cluster starts and read to execute query anytime
- no master slave architecture in Impala, each Impala node can receive client request and the `query coordinator` at each node will dispatch the task to other Impala nodes
- In short: Impala is good for PB level interactive SQL queries; Impala is a MPP database(Hive is MR based database); Impala does not have the fault tolerence as Hive does(in Hive, MR is managed by `Yarn AppMaster`, failures will be detected and rerun the task), Impala reexecutes the entire query upon failure; Impala is much faster for the query performance
- use case:
  - Impala: realtime, interactive SQL data analysis
  - Hive: large and complecated query task, offline processing
### Disadvantage of Impala
- Due to **MPP architecture** nature of Impala, an Impala cluster **can only scale up to hundreds of nodes** (whereas in Hive and MR, tens of thunds of nodes are normal). When the amount of concurrent queries is around **20**, the throughput of the entire system is full loaded, and increasing nodes can no longer increase the throughput of the cluster. Level of data amount to be handled is limited to **PB(Perabytes)**
- Impala resouces are **not managed by Yarn**, thus cannot dynamically share the cluster resources with Spark, Hive and other Hadoop components







## Q
- since number of nodes running the impala is limited, what if # of HDFS nodes >>> # of impala nodes, where do we place the `imapad` process?