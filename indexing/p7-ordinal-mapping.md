# Ordinal Mapping Creation

Create bidirectional mapping between JVector's sequential ordinals and Lucene's document IDs.

## Call Stack

```
JVectorWriter.flush()
  └─> GraphNodeIdToDocMap.build(docIds, maxDoc)
      ├─> graphNodeIdsToDocIds = docIds.toArray()
      ├─> docIdsToGraphNodeIds = new int[maxDoc + 1]
      └─> Return GraphNodeIdToDocMap
```

## Key Files

- GraphNodeIdToDocMap.java - build() entry point, mapping arrays

## Purpose

Enable O(1) translation between graph node IDs and document IDs, handling deletions and sorting.

## What Happens?

1. GraphNodeIdToDocMap.build() invoked
2. Create ordinal → docID mapping: [0,1,2,...,9999]
3. Create docID → ordinal mapping: [0,1,2,...,9999]
4. Memory: 2 × 10,000 × 4 = 80 KB
5. Both modes: Same mapping process

## Outcome

1. Bidirectional O(1) lookup
2. Handles sparse docID spaces
3. Critical for search: graph traversal → document retrieval
4. No difference between modes


## Status - Both

```java
GraphNodeIdToDocMap {
    version: 1,
    graphNodeIdsToDocIds: int[10000] {[0,1,2,...,9999]},
    docIdsToGraphNodeIds: int[10000] {[0,1,2,...,9999]},
    size: 10000,
    maxDocId: 9999,
    memoryUsage: 80,000 bytes (78.13 KB)
}
```