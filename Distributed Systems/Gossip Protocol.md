# The Problem
- How do we maintain cluster information? - Which contains all the node-partition mapping, live nodes etc.
- How does nodes know the existence of other nodes in the cluster?
- How do nodes in a cluster communicate with each other?

# The Solution
1. Centralized state management service - like ZooKeeper
2. Peer-to-peer state management

# Centralized state management service
- So basically, Zookeeper keeps information of all the live nodes and parition information in its ephemeral nodes.
- Anyone who wants this information will place a watch on this cluster information znode to be notified about any change in parition and node.
- [Read more](https://github.com/ryukirisame/learnSystemDesign/blob/main/Distributed%20Systems/2.%20Partitioning.md#approach-1-how-zookeeper-solves-the-routing-update-problem)

# Peer-to-peer state management

Here, a node communicates with other nodes to share information. We have the following ways of transferring information from one node to another:
1. point-to-point broadcast
2. eager reliable broadcast
3. gossip protocol

## Point-to-point broadcast
<img width="500"  alt="image" src="https://github.com/user-attachments/assets/9d352c06-0628-475a-937d-064f6b864bf0" />

- The producer node (The node which has got something new to share), it sends a message directly to all the nodes in the cluster in a point-to-point broadcast fashion.
- If a consumer fails to get the information, then the producer will retry with that consumer.
- But if the producer also fails, then the new information is lost forever for that consumer. The consumer will never get it.

## Eager reliable broadcast
- The idea is this: Whenever a producer sends some information to consumers, the consumers will re-broadcast to every other node in the cluster.
- So lets suppose a producer sends a message to all the N consumers. The N consumers will then re-broadcast to all the other N nodes in the cluster.
- So, for every new information, we have O(n²) messages travelling through the network.
- Pros: It solves the problem with point-to-point broadcast: If the original producer fails, that's okay, the other nodes will broadcast that information. So for a failing consumer, N retries will be done.
- Cons: High network bandwidth due to O(n²) messages.

