# HMWS Graph

Build HNSW graph using Vamana diversity algorithm, creating multi-layer structure for logarithmic search.

## Call Stack

```java
JVectorWriter.flush()
  └─> GraphIndexBuilder.build(diskMode)
      ├─> Assign vectors to layers (exponential distribution)
      ├─> Build graph layer by layer:
      │   └─> For each node:
      │       ├─> BeamSearch(ef_construction=200)
      │       ├─> VamanaDiversity(M=32, alpha=1.2)
      │       ├─> Add bidirectional edges
      │       └─> Apply degree overflow (1.2)
      └─> Parallel construction (8 threads)
          └─> DiskANN: Optimize layout for sequential disk access
```

## Key Files

- JVectorWriter.java - Graph construction entry
- JVectorIndexWriter.java - GraphIndexBuilder
- DelayedInitDiversityProvider.java - Vamana diversity

## Purpose

Transform linear vector collection into navigable graph, reducing search from O(N) to O(log N).

## What happens?

1. Layer assignment (probabilistic)
2. Neighbor selection
3. Bidirectional linking: Enables navigation in both directions
4. Degree overflow: Improves connectivity during build
5. DiskANN optimization: Layout optimized for sequential disk access


## Outcome

- HNSW graph with 5 layers
- 320,000 bidirectional edges (10,000 × 32)
- Entry point at highest layer
- Search: O(log N) ≈ 200 comparisons
- 50x speedup vs linear scan
- DiskANN: Graph layout optimized for memory-mapped access

## Status - Both

```java
OnHeapGraphIndex {
    size: 10000,
    dimension: 768,
    M: 32,
    efConstruction: 200,
    alpha: 1.2,
    diskMode: (HNSW: false, DiskANN: true),
    
    layerStructure: {
        maxLevel: 4,
        entryPoint: 9847,
        layerDistribution: {
            layer0: {nodeCount: 10000},
            layer1: {nodeCount: 5012},
            layer2: {nodeCount: 2487},
            layer3: {nodeCount: 1251},
            layer4: {nodeCount: 623}
        }
    },
    
    adjacencyLists: {
        node0: {
            layer0: {neighbors: [42,17,203,...], count: 32},
            layer1: {neighbors: [15,31,47,...], count: 32}
        }
        // ... all 10,000 nodes
    },
    
    edgeStatistics: {
        totalEdges: 320000,
        avgDegree: 32.0
    },
    
    constructionMetrics: {
        duration: 45000 ms,
        distanceComputations: 6400000
    },
    
    memoryUsage: 3,010,000 bytes (2.87 MB),
    
    diskAnnOptimizations: {
        HNSW: {contiguousStorage: false},
        DiskANN: {
            contiguousNeighborStorage: true,
            sequentialAccessOptimized: true,
            mmapFriendly: true
        }
    }
}
```
