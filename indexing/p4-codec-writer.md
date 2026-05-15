# Codec Layer - Writer Initialization

Initialize JVector codec writer, calculate PQ parameters, create file handles based on mode (combined or separate files).

## Call Stack

```
Lucene IndexWriter.addDocument()
  └─> SegmentWriter.flush()
      └─> Codec.fieldsWriter()
          └─> PerFieldKnnVectorsFormat.fieldsWriter()
              └─> JVectorFormat.fieldsWriter()
                  ├─> new JVectorWriter(state)
                  │   ├─> Detect mode from mapping:
                  │   │   ├─> HNSW: method="hnsw" → diskMode=false
                  │   │   └─> DiskANN: method="disk_ann" → diskMode=true
                  │   ├─> Calculate PQ subspaces: (int)(768 × 0.25) = 192
                  │   ├─> Create file handles:
                  │   │   ├─> HNSW mode (3 files):
                  │   │   │   ├─> meta = "_0.meta-jvector"
                  │   │   │   ├─> vectorIndex = "_0_embedding.data-jvector"
                  │   │   │   └─> neighborCache = "_0_embedding.neighbors-score-cache-jvector"
                  │   │   └─> DiskANN mode (4 files):
                  │   │       ├─> meta = "_0.meta-jvector"
                  │   │       ├─> vectorData = "_0_embedding.vec-jvector"  // Separate!
                  │   │       ├─> graphData = "_0_embedding.graph-jvector"  // Separate!
                  │   │       └─> pqData = "_0_embedding.pq-jvector"  // Optional
                  │   ├─> Write file headers
                  │   └─> Initialize thread pools (8 cores)
                  └─> Set configuration:
                      ├─> HNSW: m=32, ef_construction=200, alpha=1.2
                      └─> DiskANN: m=32, ef_construction=200, alpha=1.2, beam_width=128

```

## Key Files

- JVectorFormat.java - Creates codec writer
- JVectorWriter.java - Main writer implementation
- JVectorMethodResolver.java - Resolves PQ parameters

## Purpose

Prepare codec infrastructure for either in-memory HNSW (combined files) or DiskANN (separate memory-mappable files).

## What Happens

1. Lucene invokes JVectorFormat.fieldsWriter()
2. Create JVectorWriter with segment write state
3. Detect mode from mapping:
    - HNSW: method="hnsw" → diskMode=false
    - DiskANN: method="disk_ann" → diskMode=true
4. Calculate PQ subspaces: 192 subspaces × 4D each
5. Create file handles:
    - HNSW (3 files):
        - meta-jvector: Metadata
        - embedding.data-jvector: Graph + vectors combined
        - embedding.neighbors-score-cache-jvector: Precomputed scores
    - DiskANN (4 files):
        - meta-jvector: Metadata
        - embedding.vec-jvector: Vectors only (memory-mapped)
        - embedding.graph-jvector: Graph only (memory-mapped)
        - embedding.pq-jvector: PQ data (if applied)
6. Write file headers
7. Initialize SIMD thread pools (8 cores)
8. Set mode-specific configuration

## Outcome

- Codec writer ready for mode-specific storage
- HNSW: Combined file for vectors+graph
- DiskANN: Separate files for memory mapping
- Thread pools initialized
- PQ parameters calculated

## Status - HNSW

```java
JVectorWriter {
    configuration: {
        mode: "HNSW",
        diskMode: false,
        maxConn: 32,
        beamWidth: 200,
        alpha: 1.2
    },
    fileHandles: {
        meta: "_0.meta-jvector",
        vectorIndex: "_0_embedding.data-jvector",  // Combined
        neighborCache: "_0_embedding.neighbors-score-cache-jvector"
    },
    threadPools: {
        simdPoolMerge: ForkJoinPool(8),
        simdPoolFlush: ForkJoinPool(8)
    }
}
```

## Status - DiskANN

```java
JVectorWriter {
    configuration: {
        mode: "DISK_ANN",
        diskMode: true,
        maxConn: 32,
        beamWidth: 200,
        searchBeamWidth: 128,
        alpha: 1.2
    },
    fileHandles: {
        meta: "_0.meta-jvector",
        vectorData: "_0_embedding.vec-jvector",  // Separate!
        graphData: "_0_embedding.graph-jvector",  // Separate!
        pqData: "_0_embedding.pq-jvector"  // Optional
    },
    threadPools: {
        simdPoolMerge: ForkJoinPool(8),
        simdPoolFlush: ForkJoinPool(8)
    }
}

```