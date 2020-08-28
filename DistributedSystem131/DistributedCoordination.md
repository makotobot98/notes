### Lamport Timestamp & version vectors(Logical clocks vs vector clocks)
- Lamort Timestamp ensures the total order, but it does not ensure the total order finalization accross distributed nodes. Total order broadcast is a mechanism ensures such finalization
- version vectors is an extension of lamport timestamp, it also captures the whether two operations are concurrent or whether one is **causally dependent** on the other, whereas Lamport timestamp enforce a total ordering(from total ordering, we cannot tell if two operations are concurrent or they are causally dependent).
  - vector clocks represent an **extension of Lamport Timestamps** in that they guarantee the **strong consistency condition**
  - version vector has the problem of scalibility, the size of the vector grows linearly with the number of nodes in the cluster

## Network Partitions
- [RabbitMQ Netowork Partition Strategy](https://www.rabbitmq.com/partitions.html)

## Election Algorithms
- Election Algorithms大致有两类，一类是**Bully Election**，一类是 **Token Ring Election algorithm** https://www.jianshu.com/p/b5a77da9da38
## Question set
- for Total order broadcast, it's necessary to get all consents from all other nodes in order to keep the consistency constraints in replicated storage systems. When it comes to implement mutual exclusion(or distributed lock service), it is sufficient to only acquire majority of quorums in order to enter the critical section
- 