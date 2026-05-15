# Disk Serialization

Persist HNSW graph, vectors, and metadata to disk with mode-specific file organization.

## Call Stack
```
JVectorWriter.flush()
  └─> Write to disk files:
      ├─> HNSW mode (3 files):
      │   ├─> Write metadata (meta-jvector)
      │   ├─> Write vector index (embedding.data-jvector)
      │   │   ├─> Graph structure
      │   │   └─> Inline vectors (combined)
      │   └─> Write neighbor cache (embedding.neighbors-score-cache-jvector)
      │
      └─> DiskANN mode (3-4 files):
          ├─> Write metadata (meta-jvector)
          │   └─> Include diskMode=true, file offsets
          ├─> Write vector data (embedding.vec-jvector)
          │   └─> Vectors only (memory-mapped)
          ├─> Write graph data (embedding.graph-jvector)
          │   └─> Graph only (memory-mapped)
          └─> Optional: Write PQ data (embedding.pq-jvector)

```

## Key Files

- JVectorWriter.java - flush() writes all files

## Purpose

Make index durable and searchable, with HNSW using combined files and DiskANN using separate memory-mappable files.

## What Happens?

### HNSW

1. Metadata file (meta-jvector):
    - Header, field metadata, configuration
    - Ordinal mapping
    - Footer with checksum
2. Vector index file (embedding.data-jvector):
    - Header
    - Graph structure: layers, entry point, adjacency lists
    - Inline vectors: 10,000 × 768 floats
    - Footer with checksum
    - Combined storage: Graph + vectors in one file
3. Neighbor cache file (embedding.neighbors-score-cache-jvector):
    - Precomputed similarity scores for 320,000 edges
    - Enables faster graph traversal

### DiskANN Mode:

1. Metadata file (meta-jvector):
    - Header, field metadata
    - DiskANN config: diskMode=true, searchBeamWidth=128
    - File offsets to separate files
    - Ordinal mapping
    - Footer with checksum
2. Vector data file (embedding.vec-jvector):
    - Header
    - Vectors only: 10,000 × 768 floats
    - Sequential storage for efficient memory mapping
    - Footer with checksum
    - Memory-mapped: OS manages paging
3. Graph data file (embedding.graph-jvector):
    - Header
    - Graph only: layers, entry point, adjacency lists
    - Contiguous neighbor storage
    - Footer with checksum
    - Memory-mapped: OS manages paging
4. Optional PQ data file (embedding.pq-jvector - if PQ applied):
    - PQ codebooks and quantized vectors

## Outcome
- HNSW: combined storage
- DiskANN: separate memory-mappable files
- Both: Durable, searchable, checksummed
- DiskANN advantage: Memory-mapped access, minimal RAM

## Status - HNSW

```java
DiskState {
    mode: "HNSW",
    directory: "/data/opensearch/nodes/0/indices/a1b2c3d4/0/index/",
    
    files: {
        metadata: {
            path: "_0.meta-jvector",
            size: 102400 bytes (100 KB)
        },
        vectorIndex: {
            path: "_0_embedding.data-jvector",
            size: 31457280 bytes (30 MB),
            content: {
                graphStructure: 2560000 bytes,
                inlineVectors: 30720000 bytes  // Combined
            }
        },
        neighborCache: {
            path: "_0_embedding.neighbors-score-cache-jvector",
            size: 1280000 bytes (1.2 MB)
        }
    },
    
    totalDiskUsage: 32839680 bytes (31.32 MB),
    
    memoryUsage: {
        onOpen: ~33000000 bytes (31.5 MB),  // Loaded into RAM
        duringSearch: ~33000000 bytes
    }
}
```

### Status - DiskANN

```java
DiskState {
    mode: "DISK_ANN",
    directory: "/data/opensearch/nodes/0/indices/a1b2c3d4/0/index/",
    
    files: {
        metadata: {
            path: "_0.meta-jvector",
            size: 102400 bytes (100 KB),
            memoryMapped: false
        },
        vectorData: {
            path: "_0_embedding.vec-jvector",
            size: 30720000 bytes (30 MB),
            memoryMapped: true  // OS manages paging
        },
        graphData: {
            path: "_0_embedding.graph-jvector",
            size: 2560000 bytes (2.5 MB),
            memoryMapped: true  // OS manages paging
        }
    },
    
    totalDiskUsage: 33382400 bytes (31.84 MB),
    
    memoryUsage: {
        metadataInRAM: 102400 bytes,
        vectorDataMapped: 30720000 bytes (OS managed),
        graphDataMapped: 2560000 bytes (OS managed),
        activeWorkingSet: ~500000 bytes (0.5 MB),
        totalRAMFootprint: ~600000 bytes (0.6 MB)  // 55x less!
    }
}
```