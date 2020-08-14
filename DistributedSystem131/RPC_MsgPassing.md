# Message Passing

## Socket based(TCP/UDP) and MPI
- message passing mechanisms requiring both sender and receiver be up and active during the time of communicating
- target transient message passing, sender and receiver are timely coupled
- three patterns:
  - request-reply
  - publish-subscribe
  - pipeline (only one subscriber gets to poll the message)

## messgae-queuing systems/Message-oriented Middleware(MOM)
- offer intermediate-term storage capacity for messages, without requiring either sender or receiver to be active during message transmission
- permits communication to be loosely coupled in time

### Message Broker
- a message queue system is subjected with many different sender and receivers with different requirement for semantics of the messages. A **message broker** acts as an application-level gateway in a message-queuing system, its main purpose is to convert incoming messages so that they can be understood by the destination application
  > A message broker can be as simple as a reformatter for messages
- in a **publish subscribe** model, a broker is responsible for **matching** applications based on the messages that are being exchanged.
  > in a **publish subscribe** model, applications send messages in the form of **publishing**. Applications publish a message on topic X, then broker is responsible for matching the message and send to those who subscribed to topic X