# Flush Trigger & Product Quantization

Flush accumulated vectors, optionally apply PQ compression if vector count exceeds threshold.

## Call Stack

```java
JVectorWriter.flush()
  └─> FieldWriter.flush()
      ├─> Check: vectorCount < 25,000?
      │   └─> 10,000 < 25,000 → Skip PQ (both modes)
      ├─> Create score provider:
      │   └─> new RandomAccessScoreProvider(vectorValues, similarityFunction)
      │       ├─> HNSW: similarityFunction = COSINE
      │       │   └─> distanceFunction: (v1, v2) -> 1.0 - dotProduct(v1, v2)
      │       └─> DiskANN: similarityFunction = EUCLIDEAN
      │           └─> distanceFunction: (v1, v2) -> sqrt(sum((v1[i]-v2[i])^2))
      └─> Ready for graph construction

Alternative (if >= 25,000 vectors):
  └─> Train PQ, quantize vectors, create PQBuildScoreProvider
      └─> DiskANN: Write PQ data to separate .pq-jvector file

```

## Key Files

- JVectorWriter.java - flush() entry point

## Purpose

Compress vectors using PQ (14-32x) for large datasets, or use full precision for smaller datasets.

## What Happens?

1. Flush triggered by segment commit
2. Check vector count: 10,000 < 25,000 → Skip PQ (both modes)
3. Create score provider:
    - HNSW: COSINE distance = 1.0 - dotProduct(v1, v2)
    - DiskANN: L2 distance = sqrt(sum((v1[i]-v2[i])^2))
4. Score provider ready for graph construction

If ≥25,000 vectors:

1. Train PQ codebooks: 192 subspaces × 256 centroids
2. Quantize: 768 floats → 192 bytes (32x compression)
3. HNSW: PQ data kept in memory
4. DiskANN: PQ data written to separate .pq-jvector file

## Outcome

- Small dataset: Full precision (both modes)
- Large dataset: 14-32x compression via PQ
- DiskANN: PQ data in separate memory-mappable file
- Score provider ready for graph construction

## Status (No PQ - Small Dataset)

```java
FlushState {
    vectorCount: 10000,
    pqApplied: false,
    
    buildScoreProvider: RandomAccessScoreProvider {
        vectorValues: ListRandomAccessVectorValues {vectors: <10,000>, dimension: 768},
        similarityFunction: (HNSW: COSINE, DiskANN: EUCLIDEAN),
        distanceFunction: {
            HNSW: (v1, v2) -> 1.0 - dotProduct(v1, v2),
            DiskANN: (v1, v2) -> sqrt(sum((v1[i]-v2[i])^2))
        }
    },
    
    vectorStorage: {
        type: "FULL_PRECISION",
        bytesPerVector: 3072,
        totalSize: 30,720,000 bytes,
        compressionRatio: 1.0
    }
}
å
```

## Alternative Status (PQ Applied - Large Dataset)

```java
FlushState {
    vectorCount: 25000,
    pqApplied: true,
    
    pqTraining: {
        duration: 30000 ms,
        configuration: {
            M: 192,  // Subspaces
            K: 256,  // Clusters per subspace
            subspaceDimension: 4,
            algorithm: "K_MEANS_PLUS_PLUS"
        },
        codebooks: float[192][256][4],
        memorySize: 786,432 bytes
    },
    
    pqVectors: {
        compressedVectors: byte[25000][192],
        originalSize: 76,800,000 bytes,
        compressedSize: 5,586,432 bytes,
        compressionRatio: 14.4x
    },
    
    buildScoreProvider: PQBuildScoreProvider {
        pqVectors: <above>,
        similarityFunction: COSINE
    }
}
```
