# Vector Accumulation in Memory

Accumulate vectors in memory as documents are indexed, storing them temporarily in ArrayList.

## Call Stack

```java
JVectorWriter.addValue(docID, vectorValue)
  └─> getFieldWriter(fieldInfo)
      ├─> First call: create new FieldWriter
      │   └─> new FieldWriter(fieldInfo, segmentName, diskMode)
      │       ├─> vectors = new ArrayList<VectorFloat<?>>()
      │       ├─> docIds = new ArrayList<Integer>()
      │       └─> diskMode = (HNSW: false, DiskANN: true)
      └─> fieldWriter.addValue(docID, vectorValue)
          ├─> vectors.add(VectorFloat.wrap(vectorValue))
          ├─> docIds.add(docID)
          └─> Track memory: 10,000 × 768 × 4 = 30,720,000 bytes
```

## Key Files

- JVectorWriter.java - addValue() entry point

## Purpose

Buffer vectors in RAM until flush threshold reached, enabling efficient batch processing.

## What Happens?

1. First vector triggers FieldWriter creation with diskMode flag
2. Each vector added via addValue(docID, vectorValue)
3. Vectors stored in ArrayList<VectorFloat<?>>
4. DocIDs stored in parallel ArrayList<Integer>
5. Memory tracked: 10,000 × 768 × 4 = 30.72 MB
6. Continue until flush triggered
7. Both modes: Same accumulation process

## Outcome
- 10,000 vectors accumulated in RAM
- Parallel docID array maintains mapping
- Memory usage: 30.76 MB
- Ready for flush
- No difference between modes at this phase


## Status - Both

```java
JVectorWriter {
    fields: [
        FieldWriter {
            fieldInfo: {name: "embedding", dimension: 768},
            segmentName: "_0",
            lastDocID: 9999,
            diskMode: (HNSW: false, DiskANN: true),
            vectors: ArrayList<VectorFloat<?>> {
                size: 10000,
                elements: [
                    VectorFloat([...]),  // doc 0
                    VectorFloat([...]),  // doc 1
                    ...
                    VectorFloat([...])   // doc 9999
                ]
            },
            docIds: ArrayList<Integer> {
                size: 10000,
                elements: [0, 1, 2, ..., 9999]
            },
            memoryUsage: 30,760,016 bytes (29.33 MB)
        }
    ]
}
```