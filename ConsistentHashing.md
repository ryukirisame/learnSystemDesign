# Consistent Hashing
- When the load on servers increase, we scale our system. If we decide to do horizontal scaling, we add new servers so that each server handles load uniformly.
- For this to work, it is important that the data is also distributed uniformly.
- Consistent hashing is a common technique to achieve this goal.

# Problems with Naive Consistent Hashing (without Virtual Nodes)
#### Naive Consisten Hashing assumes the following:
1. The hash function is good enough to distribute node evenly across the ring. i.e., the nodes are placed on the ring at equal distances (equal partition of hash space).
2. Servers have equal capacity.

#### The problems:
1. Chain of failures: If server B fails, all the traffic of B will go to C. C will not be handle twice the traffic now and it will fail. Now, all the traffic from B, C and D will go to D. D will also fail. And so on...like a chain of failures/cascading effect.
2. If the hashing function places nodes un-evenly on the ring, some servers will take less load, while some will have more load.
3. If nodes are added or removed from the node, the partition/hash space of some servers will be too large, or too small. 

#### Solution
- To mitigate the problems listed above, we use virtual nodes.
- Virtual nodes are nodes that refer to actual nodes.
- We place multiple virtual nodes for the same server on the ring. The more the virtual nodes, the more balanced the distribution of keys among the servers. 

### Implementation

```java
class ConsistentHashing<T> {

    TreeMap<Integer, T> circle = new TreeMap<>();
    int numberOfReplicas; // Number of virtual nodes for each node
    HashFunction hashFunction;

    ConsistentHashing(HashFunction hashFunction, int numberOfReplicas, T[] nodes){
        this.hashFunction = hashFunction;
        this.numberOfReplicas = numberOfReplicas;
        
        // Put all the nodes on the circle 
        for(T node: nodes){
            addNode(node);
        }
    }


    void addNode(T node){
        // We put node in the circle 'numberOfReplicas' times
        for(int i = 0;i<numberOfReplicas;i++){

            Integer hash = hashFunction.hash(node.toString() + i);
            circle.put(hash, node);
        }

    }

    void removeNode(T node){
        for(int i =0; i<numberOfReplicas; i++){
            Integer hash = hashFunction.hash(node.toString() + i);
            circle.remove(hash);
        }
    }

    T getNode(Object key){
        
        // No nodes exists on the circle
        if(circle.isEmpty()){
            return null;
        }
        
        // Find hash of the key
        Integer hash = hashFunction.hash(key);

        // get the tail of the map starting from the 'hash' key
        SortedMap<Integer, T> tail = circle.tailMap(hash);
    
        // Hash of the node where 'key' is stored
        Integer nodeHash = null;

        // If there are no nodes adjacent to 'hash', the request should go to the first node
        if(tail.isEmpty()){
            nodeHash = circle.firstKey();
        }
        // If there is a node after 'hash', the request should go the next available node
        else{
            nodeHash = tail.firstKey();
        }

        return circle.get(nodeHash);
        
    }
    
}
```

- Note:  Consisten hashing doesn't eliminate redistribution of data among servers upon horizontal scaling, it minimizes it.
