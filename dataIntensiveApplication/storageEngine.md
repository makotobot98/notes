# Storage Engine

## Index
**well-chosen indexes speed up read queries, but every index slows down writes**, any kind of index slows down writes, because the index also needs to be updated every time data is written. Index improves the read query performance

## Bitcask
- In memory hash indexes(all keys fit in the availible RAM, since the hashmap is kept completely in memory). The values can use more space than there is availible in memory, since they can be loaded from disk with **just one disk seek**. If that part of the data is already in the filesystem cache, a read doesn't require any disk I/O at all
- high-performance reads and writes
- suited to situations where the value for each key is updated frequently, all keys are not so distinct so they fit in memory