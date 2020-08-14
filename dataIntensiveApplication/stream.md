# Streams and Stream Processing


## Message Broker

- log structured message broker

## CDC
- use case: ensures the consistency states accross different **derived data systems** (e.g., full index search database and data warehouse)
- background compaction, use tombstone to determine the final state of a record(deleted or modified to a final deterministic value)

## Event Sourcing
- higher level abstraction than CDC, cares about not just the final state of a record but rather, the process of that transformation. Such Event Sourcing can be better to learn the application
 - e.b., for an ship object (like a ship that sails over the seas from place to place), say the ship traveled three places from `A -> B -> C`. typically in CDC, the final state of the ship is `C`, but with Event Sourcing, we can also see the intermediate state `B`