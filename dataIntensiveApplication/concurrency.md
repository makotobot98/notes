# Consistency
## ACID
- atomic: operations are either commited or aborted, leaving no half state/partial failures
- isolation: different operations wont step on each other's toes
- durability: data are persisted to SSD/or replicated accross nodes
- consistency: depends on application level of consistency

## some common miconceptions
- some databases operations are not necessarily **atomic**. e.g., an update() in a relational database might involve multiple updates due to the foreign key constriants, and maybe some updates are successful and some failed, leaving the database in a inconsistant state
## The mindset dealing with concurrency problems
Having complete ACID guarantee for databases is almost too expensive as to ensure serializibility, the performance penalty is too costly. It's therefore common for systems to use weaker levels of isolation, which protect against some concurrency issues, but not all. Rather than blindly replying on tools, we need to develop a good understanding of the kinds of concurrency problems that exist, and how to prevent them. Then we can build applications that are reliable and correct, using the tools at our disposal.

## isolation levels
### Read Commit
- keep two versions to avoid dirty read / dirty write
### Snapshot isolation
- MVCC to allow long-running backups, reader does not block writers
- guarantees read-only transactions only see non-concurrenct(old writes using increasing transaction id) writes
- does not prevent **Lost Update**
### Ways to prevent Lost Update
#### single leader-based database
- **Explicit locking** on the rows to be read, to finish the read-update cycle (which blocks other readers on the same row)
- **Compare-and-set**
> **Explicit Locking** and **Compare-and-Set** works only for single leader-based, and does not work for multi-leader or leaderless replication. Due to the fact that it assumes that we are reading and then locking the single up-to-date copy of the data.
>
> databases with multi-leader or leaderless replication usually allow several writes to happen concurrently and replicate them asynchronously, so they cannot guarantee that there is a single up-to-date copy of the data

### Phantom
- `phantom` refers to a write in one transaction changes the result of a search query in another transaction.
  - `phantom` cannot always be resolved using locks on the row, since it might:
    - involve multiple rows or entire table(well, yes still can use locks in this case to lock the entire table)
    - involve write to an new object, which we cannot lock something that does not preexists

### Serializable isolation
Serializable isolation is the strongest isolation level. It guarantees that even though transactions may execute in parallel, the end result is the same as if they had executed one at a time, serially, without concurrency. Thus the database guarantees that if the transactions behave correct when run concurrently - in other words, the database prevents all possible race conditions

Type of Serializable isolation
- complete serial execution
- Two-Phase Locking (readers block writers, writers block readers)
  - huge overhead of acquiring and releasing all those locks, reduced concurrency
  - two phase locking resolves all read,write skews discussed before. (for phantom can be resolved using predicate locks which locks the condition of the search query, so non-preexistent record can be locked too)
- serializable snapshot isolation