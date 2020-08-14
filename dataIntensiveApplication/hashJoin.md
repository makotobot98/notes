# [Hash Joins](https://en.wikipedia.org/wiki/Hash_join)
terms:
when joining relation R and S and given `|R| < |S|`, **building relation**(R) is usually refers to the **smaller relation where we try to fit the building relation in memory to form a hash table**, and by **scanning the probing relation**(S) which is the **bigger** relation to output the result joined tuples.
## Classic Hash Join

## [Grace Hash Join](https://www.youtube.com/watch?v=GRONctC_Uh0)
- partition relation R and S, where its guaranteed that the tuples Ri and Si that are of same key must be assigned(through a hash function) to the same partition(a partition can be one or multiple nodes), then perform the classic hash on each partition
- process: scan relation R and put rows of R into a buffer which maps to a specific partition, when buffer is full, buffer is flushed onto disk with assigned partition
- the key is to ensure each partition **can load all building tuples in memory**
- cannot handle data skew where one partition is assined with too many tuples

> Q: what if a partition R cannot all fit in memory pages?: keep partition R until can fit