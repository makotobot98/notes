# Batch
### Why do we need Batch?
- large ad hoc query in parallel could easily overwhelm the database (this is why we need OLTP and OLAP system)
- Making random-access requests over the network for every record u want to process is slow
- querying a remote database would mean that the batch job becomes **nondeterministic**, because the data in the remote database might change while the job is running. **So we need to take a copy/snapshot of the user database backup using an ETL process**, and put it in the distributed filesystem

## Reduce-side Joins and Grouping with MapReduce
### Sort-merge join
- as the name suggested, when joinin two tables rows of two tables that belong to the same key is mapped to the same key during the `mapper`, then `reducer` merge the sorted lists of the records from both sides of the join
### Group By
```sql
group by s.studentId
avg(s.studentGrade)
```
- group by is similar where in SQL we do a `group by` and then do some aggregations. In **MapReduce** we can accomplish by mapping all rows by `groupBy ID` then do the aggregation logic during the `reducer`
- when doing group by on **skewed data**, when can split the job into multiple stages and assign the `hot key` a fake id(which is the hash id that each record gets mapped on, application must remember what those ids are), and through multiple jobs we can finish the group by in better parallelism
### Skewed join
- in the case where the table is too large related to a single key(`hot keys`), we have to perform the join that handles this data skew. (as all nodes in `MapReduce` will have to wait for one single node to finish the job which slows down the job flow, **recall in mapreduce all records belonging to the same key will be mapped to the same partition which is assigned to a single node, in this case if the records are too large and cannot fit all in memory, the remaining records will be placed in the disk**)
- there are some strategies for skewed joins:
  - **skewed join implemented by Pig**: first run a sampling job to determine the hot keys, when perform the actual join, `mappers` send records relating to a hot key to one of the several reducers choosen at random(**so we handle the hot key by spreads the work over several reducers to allow better parallelism**), `hot key records need to be replicated to all reducers handling that key`([Practical Skew Handling in Parallel Joins[1992]](http://www.vldb.org/conf/1992/P027.PDF)) 
  - **skewed join implemented by Hive**: require hto keys to be specified explicitly in the table metadata, and it stores records related to those keys in seperate files from the rest

## MapReduce
### pros
- fault tolerent: each job(and usually a workflow consists of many jobs) output is written to the disk, and thus MapReduce is suitable for the multi-tenent system(cloud) with frequent task termination, since intermediate jobs are written to disks which are durable; failed jobs can be retried easily.
- good abstraciton: only contact point of a job is the input/output distributed file directory, loose coupling; `mapper` and `reduer` model which can be implementing many other workflows (joins, aggregations, etc..)
### cons
- each job output is written to the disk, meaning some type of workflow will be extremely slow since some jobs' output are not needed to be sent to the disk due to those job outputs are only **intermedediate state**(a means of passing data from one job to the next); e.g., recommandation systems usually consists of 50-100 MapReduce Jobs, which there are a lot of such intermediate state. This process of writing out this intermediate state to files is called **materialization**
  - **Materialization** caused MapReduce job can only start only when all tasks in the preceding jobs are completed. (`Recall MapReduce Jobs input is immutable, and output is completely replaced`). Thus a skew or varying loads can make a job significantly slow
  - Storing the intermediate state in a distributed filesystem means those intermediate state files are also **replicated**
- `Mapper` depends on the situations can be **redundent** (recall in MapReducer we **MUST** have mappers, while reducers can be **ommited**), since within a chaining of jobs, the input for one job's mapper could just be the reducer from last job, meaning the data was just written by the previous reducer must be read again by the mapper, this increases time in the Disk IO
- 