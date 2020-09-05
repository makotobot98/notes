## Consistency vs Coherence
- A consistency model defines ordering of accesses to all data items in the data store
- Coherence defines what happens on access to a single data item

### Consistency Rank(descending):
- Linearizability: maintains a global, real-time ordering
- Sequential Consistency: ensures at least a global ordering (not real time, as long as all nodes aggree on the ordering, one node can lag behind a few operations behind another node)
- Causal Consistency: ensures partial orderings between dependent operations (preserve causal dependencies, does not ensure the order for concurrent dependency, relaxes sequential consistency requirement while ensures causality and improves performance)
   > causality only incurrs when there are a read following a write. two writes are concurrent if one write did not read the other write before writing
   - Writes that are potentially causally related must be seen by all processes in the same order. 
   - Concurrent writes may be seen in a different order by different processes.
- Entry Consistency: consistency model built on top of `grouping operators`(distributed locking, synchronization mechanism)
- Eventual Consistency

## Cache Coherence
### Cache State Protocol
- MSI, MESI, MOESI
### Mechanisms
- Snooping
- Directory-based


## Client centric consistency
- monotonic read
- monotonic write
- read-your-write consistency
- write-follow-read consistency


# Q
- monotonic write & monotonic read


