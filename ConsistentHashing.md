# Consistent Hashing
- When the load on servers increase, we scale our system. If we decide to do horizontal scaling, we add new servers so that each server handles load uniformly.
- For this to work, it is important that the data is also distributed uniformly.
- Consistent hashing is a common technique to achieve this goal.




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
