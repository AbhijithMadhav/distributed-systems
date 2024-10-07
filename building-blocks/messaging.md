# Asynchronous messaging

Messaging brokers help with asynchronous communication between components.
They encourage a event-driven model of workflow execution which decouples components in a system.
This decoupling means that failure of one component does not affect the others.

Expectations from messaging brokers
* High throughput processing
* Low latency processing
* Horizontally scalable
* Fault tolerant

Messaging brokers are usually distributed clusters.

## Kafka
Kafka offers high throughput, low latency processing via partitioning.
Partitioning is a measure of the write/read concurrency in kafka.

Horizontal scalability is via shared-nothing architecture. 
Addition of more broker nodes to the cluster increases compute and storage.
It is to be noted that it was not possible to scale only storage without compute until recently.

Shared nothing architecture also helps with fault tolerance. 
No failure of a single node can bring the entire system to a halt.

Fault tolerance is also aided by replication of partitions.

Write conflicts are avoided by using a single leader replication model.

Leader elections are done via the Raft protocol.

## Operational difficulties
Increasing partitions of existing topics
Adding new brokers means migrating replicas and replica leaders

