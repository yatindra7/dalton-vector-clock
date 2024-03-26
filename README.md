# Making an efficient, and dynamic vector clock implementation for DALTON

## Vector clocks, and Lamport clocks

[Lamport Clocks](https://amturing.acm.org/p558-lamport.pdf) define a single total ordering of events by having each process maintain an integer value, initially zero, which it increments after every atomic event. These local logical clocks that are incremented per process are synchronized with one another by sending the current local logical clock value on every outgoing message. When a peer in the system receives such a message, it sets its own local logical clock to be greater than this value, if it is not already.

![image](https://github.com/yatindra7/dalton-vector-clock/assets/84220034/1cda7cb1-671c-45a2-b08e-f3a1cabe9da4)

This simple algorithm maintains consistency among the peers since the sending of a message is always set to “happen before” the receipt of the message. In Lamport clocks only one possible ordering and knowledge of any other ordering is lost. 

For problems that require knowledge of the global program state, the approach of retaining all possible orders is handled with [vector clocks](https://en.wikipedia.org/wiki/Vector_clock). Vector clocks are a logical data structure that represent the partial order of events in a distributed system. They consist of a vector of counters, one for each process or node in the system, that are incremented whenever a process performs an event or sends a message. The vector clock of a process is attached to every message it sends, and updated with the maximum of its own and the received vector clocks. This way, vector clocks capture the causal dependencies and the potential concurrency of events in the system. To generalize the Lamport Clock algorithm to handle all possible orderings, vector clocks represent timestamps as an array with an integer clock value for every process in the network.

## Partial ordering property in vector clocks

![image](https://github.com/yatindra7/dalton-vector-clock/assets/84220034/a2ee9651-50fe-45be-8ccd-9959c5648739)

## Scaling vector clocks

Vector clocks grow linearly with the number of processes or nodes in the system. This means that they can consume a lot of bandwidth, storage, and computation resources, especially for large-scale distributed systems. To scale vector clocks, you can use some techniques such as compression, pruning, aggregation, or approximation. For example, you can **compress vector clocks by using bitmaps, delta encoding, or sparse vectors**.

Most importantly for our work with DALTON, we can **prune vector clocks by removing obsolete or irrelevant counters**. An important implementation example to follow is that of [Riak](https://riak.com/posts/technical/vector-clocks-revisited/index.html?p=9545.html).

## Proving correctness of a vector clock implementation

Proving the correctness of vector clocks, involves demonstrating that they accurately capture the causal relationships between events in a distributed system. This is crucial for applications such as debugging and recovery from failures. The correctness of vector clocks is typically established through mathematical proofs that show they satisfy the necessary conditions for causality tracking.

We intend to go about this using an **inductive approach**, and for the **specific scenario of DALTON**, where **in-order (FIFO) delivery is ensured**/assumed using a centralised queue system. The inductive proof technique is exemplified in this literature on [using induction to prove properties of distributed programs](https://citeseerx.ist.psu.edu/document?repid=rep1&type=pdf&doi=e79a4ec0c15296eae91a734671505251d41143fb).

And as mentioned by the [mixed-vector clock paper](https://arxiv.org/pdf/1901.06545.pdf), to prove the correctness of our implementation, we need to show that it satisfies the constraint:

![image](https://github.com/yatindra7/dalton-vector-clock/assets/84220034/ef854d34-c5ff-45a3-8d68-962853d442e2)


```
∀s, t : s → t ⇔ s.v < t.v.
(⇒) s → t ⇒ s.v < t.v
```

## Distributed systems using vector clocks

### Amazon DynamoDB
DynamoDB, a fully managed NoSQL database service by Amazon, uses vector clocks to track versioning and reconcile conflicting updates among replicas across multiple data centers. 

DynamoDB maintains a vector clock associated with each item stored in the database. This vector clock consists of a list of (node, counter) pairs, where each pair represents the version of the item observed by a particular node. During updates, DynamoDB increments the counter corresponding to the local node and merges vector clocks when reconciling conflicting updates from different nodes.

### Riak
Riak, a distributed NoSQL database, uses vector clocks to track the causal relationships between different versions of key-value pairs stored across its cluster. Each object in Riak is associated with a vector clock, which is updated on each write operation. During reads, Riak compares the vector clock of the requested object with the vector clocks of other replicas to determine causality and resolve conflicts.

### Google's Bigtable

Bigtable, a distributed storage system used by Google for large-scale data processing,  employs vector clocks to maintain versioning and consistency among replicas of data cells stored in its distributed storage system. Each data cell in Bigtable is associated with a vector clock, which is updated whenever the cell is modified. Vector clocks are used to detect conflicts and reconcile concurrent updates during read and write operations.

### Apache Cassandra

Cassandra, a highly scalable distributed database management system, utilizes vector clocks to track the versioning and ordering of updates to data partitions distributed across its cluster. Each row in Cassandra is associated with a vector clock, which is updated on each write operation. During reads, Cassandra compares vector clocks to determine the causal relationships between different versions of the data and resolve conflicts.

### Git

Git, a distributed version control system, uses vector clocks (known as commit hashes) to uniquely identify different versions of files and directories within a repository. Each commit in Git includes a cryptographic hash of its content and parent commits, forming a directed acyclic graph that represents the history and causality of changes in the repository.

### Amazon S3 (Simple Storage Service)

S3, a scalable object storage service provided by Amazon Web Services, employs vector clocks to manage versioning and consistency among replicas of objects stored across multiple data centers. Each object version is associated with a timestamp and a unique identifier, which helps S3 track the ordering of updates and resolve conflicts during concurrent writes.

### Apache Kafka

Kafka, a distributed streaming platform, uses offsets (similar to sequence numbers) to establish ordering guarantees and ensure message delivery consistency across partitions. Each message in Kafka is assigned a unique offset, which is used to track the sequence of messages within a partition and detect inconsistencies or message reordering.

### Apache HBase

HBase, a distributed, scalable, and consistent NoSQL database built on top of Hadoop, employs vector clocks to maintain consistency and versioning among replicas of data stored in its distributed tables. Each cell in an HBase table is associated with a vector clock, which is updated on each write operation. Vector clocks help HBase track the causal relationships between different versions of data and resolve conflicts during reads and writes.

### Redis

Redis, an in-memory data structure store, can be configured to use vector clocks for conflict resolution in its distributed implementation (Redis Cluster). Vector clocks help Redis ensure eventual consistency and resolve conflicts among replicas of distributed data structures such as sets, lists, and sorted sets.

### Couchbase

Couchbase, a distributed NoSQL database, utilizes vector clocks to maintain consistency and versioning across replicas of data stored in its distributed key-value store. Each document in Couchbase is associated with a vector clock, which is updated on each write operation. Vector clocks help Couchbase track the causal relationships between different versions of documents and resolve conflicts during reads and writes.

## Algorithm

### Initialization
Each node in the DALTON system starts with a vector clock of size N (where N is the number of nodes), initialized to zero, except it's own counter is set to 1.

### Event Handling
#### Internal Event
When an internal event occurs at a node, the node increments its own component in the vector clock by 1

#### Send Event
When a node sends a message, it increments its own component in the vector clock by 1 and sends the updated vector clock to the receiver.

#### Receive Event
Upon receiving a message, a node updates its vector clock by taking the maximum of its current vector clock and the received vector clock for each component.

#### Adding a Node
The new node will have other nodes' counters initialized to 0, and it's own initialized to 1.

#### Pruning Inactive Node
In the night timeun a seperate algorithm where the nodes come to consensus on a removal of such a node.

### Pruning Inactive Nodes
Every 24 hours, the system prunes out inactive nodes from its vector clocks. This is done by removing the components corresponding to inactive nodes from the vector clocks of all active nodes.

### Formal Notation

Let $V_i$ denote the vector clock of node (i), where $V_i = [v_{i1}, v_{i2}, ..., v_{iN}]$ and $v_{ij}$ represents the local logical clock of node (i) for node (j).

- Internal Event: $V_i[i] := V_i[i] + 1$

- Send Event: $V_i[i] := V_i[i] + 1$ and send $V_i$ to the receiver.

- Receive Event: Upon receiving $V_j$, update $V_i$ as follows: $V_i[k] := \max(V_i[k], V_j[k])$ for all $k$

- Adding a Node: Let $V_i$ be the vector clock of node $i$, and $N$ be the total number of nodes. When a new node $j$ is added, update $V_i$ for all $i$ as follows: $V_i := V_i \cup [0]$, where $V_i \cup[0]$ denotes the vector clock of node $i$ with an additional component of zero at the end.

- Pruning Inactive Node: Let $I$ be the set of inactive nodes. For each node, update $V_i$ as follows: $V_i := V_i \setminus I$, where $V_i \setminus I$ denotes the vector clock of node $i$ with the components corresponding to inactive nodes removed.

### Assumption
- FIFO Delivery: The algorithm assumes FIFO delivery of messages, which is a fundamental requirement for vector clocks to function correctly. This ensures that messages are received in the order they were sent, allowing for accurate updates of vector clocks upon receiving messages.
- Asynchronous Communication: The algorithm accommodates asynchronous communication by allowing for delays in message delivery. This is inherent in the design of vector clocks, which do not rely on the timing of message delivery but rather on the logical order of events.

### Proof of Correctness

- Causality: The vector clocks provide a partial ordering of events, ensuring that if event $e_i$ causally precedes event $e_j$, then $V_{i} < V_{j}$. This is guaranteed by the rules for updating vector clocks upon internal, send, and receive events.
- Consistency: The system maintains a consistent view of the global logical time across all nodes, as each node's vector clock accurately reflects the causal relationships between events.
- Pruning: Pruning inactive nodes ensures that the vector clocks remain compact and relevant, without including information about nodes that are no longer active. This does not affect the causality and consistency properties of the vector clocks.
