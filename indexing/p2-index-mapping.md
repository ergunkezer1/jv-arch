# Index Mapping

Parse user-provided index mapping and create configured field mapper with resolved parameters for either HNSW or DiskANN mode.

## Call Stack

```
REST API: PUT /my-index
  └─> RestCreateIndexAction.prepareRequest()
      └─> MetadataCreateIndexService.createIndex()
          └─> MapperService.merge()
              └─> DocumentMapper.parse()
                  └─> KNNVectorFieldMapper.TypeParser.parse()
                      ├─> parseMapping(context, fieldNode)
                      │   ├─> Extract: dimension, method, space_type, engine, parameters
                      │   │   HNSW: method="hnsw" or omitted (default)
                      │   │   DiskANN: method="disk_ann" (explicit)
                      │   └─> Validate: dimension > 0, valid method, valid space type
                      ├─> SpaceType.getSpace(space_type)
                      │   ├─> HNSW: "cosinesimil" → COSINESIMIL → COSINE
                      │   │   SpaceType.java - COSINE requires normalization
                      │   └─> DiskANN: "l2" → L2 → EUCLIDEAN
                      │       L2 works best with DiskANN, no normalization needed
                      ├─> KNNEngine.getEngine(engine)
                      │   └─> "jvector" → KNNEngine.JVECTOR
                      │       KNNEngine.java - Same engine, different modes
                      ├─> JVectorMethodResolver.resolveMethod()
                      │   ├─> HNSW mode:
                      │   │   ├─> Fill defaults: m=32, ef_construction=200
                      │   │   └─> Add: alpha=1.2, neighbor_overflow=1.2
                      │   │       Standard HNSW parameters
                      │   └─> DiskANN mode:
                      │       ├─> Fill defaults: m=32, ef_construction=200
                      │       ├─> Add: alpha=1.2, beam_width=128
                      │       └─> Set: disk_mode=true
                      │           JVectorDiskANNMethod - On-disk storage enabled
                      └─> LuceneFieldMapper.Builder.build()
                          ├─> Create validators:
                          │   ├─> FiniteValueValidator (both modes)
                          │   ├─> DimensionValidator(768) (both modes)
                          │   └─> SpaceTypeValidator
                          │       ├─> HNSW: SpaceTypeValidator(COSINESIMIL) - rejects zero vectors
                          │       └─> DiskANN: SpaceVectorValidator(L2) - no zero check
                          ├─> Create transformer:
                          │   ├─> HNSW: NormalizeVectorTransformer (L2 normalize for COSINE)
                          │   └─> DiskANN: IdentityTransformer (no transformation for L2)
                          └─> Build LuceneFieldMapper

```

## Key Files

Key Files:

- KNNVectorFieldMapper.java:326 - Entry point
- SpaceType.java - Space type resolution
- JVectorDiskANNMethod.java - DiskANN configuration
- NormalizeVectorTransformer.java - HNSW normalization
- LuceneFieldMapper.java - Field mapper

## Purpose

Transform user mapping configuration into internal field mapper optimized for either in-memory HNSW or on-disk DiskANN storage.

## What Happens?

1. User submits mapping via REST API: PUT /my-index with knn_vector field
2. KNNVectorFieldMapper.TypeParser.parse() invoked
3. Mode Detection:
    - HNSW: method="hnsw" or omitted → standard in-memory mode
    - DiskANN: method="disk_ann" → on-disk memory-mapped mode
3. Space Type Resolution:
    - HNSW: "cosinesimil" → COSINE (requires normalization)
    - DiskANN: "l2" → EUCLIDEAN (no normalization)
4. Parameter Resolution:
    - HNSW: m=32, ef_construction=200, alpha=1.2, neighbor_overflow=1.2
    - DiskANN: m=32, ef_construction=200, alpha=1.2, beam_width=128, disk_mode=true
5. Validator Creation:
    - Both: FiniteValueValidator, DimensionValidator(768)
    - HNSW: SpaceTypeValidator(COSINESIMIL) - rejects zero vectors
    - DiskANN: SpaceVectorValidator(L2) - no zero vector check
6. Transformer Creation:
    - HNSW: NormalizeVectorTransformer (L2 normalize for COSINE)
    - DiskANN: IdentityTransformer (no transformation for L2)
7. Build LuceneFieldMapper with all components
8. Store mapping in cluster state

## Outcome

- Field mapper created with complete configuration
- Validators ensure data quality at ingestion time
- Transformer prepares vectors for COSINE similarity computation
- Mapping stored in cluster state and persisted
- Index ready to accept documents with vector fields

## Status - HNSW

```java
IndexMetadata {
    index: "my-index-hnsw",
    mappings: {
        "embedding": LuceneFieldMapper {
            name: "embedding",
            fieldType: KnnFloatVectorField.FieldType {
                dimension: 768,
                vectorEncoding: FLOAT32,
                similarityFunction: COSINE  // HNSW with COSINE
            },
            perDimensionValidator: FiniteValueValidator,
            vectorValidator: [
                DimensionValidator(768),
                SpaceTypeValidator(COSINESIMIL)  // Rejects zero vectors
            ],
            vectorTransformer: NormalizeVectorTransformer,  // L2 normalization
            knnMethodContext: {
                engine: JVECTOR,
                spaceType: COSINESIMIL,
                methodComponentContext: {
                    name: "hnsw",
                    parameters: {
                        m: 32,
                        ef_construction: 200,
                        alpha: 1.2,
                        neighbor_overflow: 1.2
                    }
                }
            }
        }
    }
}
```

## Status - DiskANN

```java
IndexMetadata {
    index: "my-index-diskann",
    settings: {
        "index.knn.disk_ann.enabled": true
    },
    mappings: {
        "embedding": LuceneFieldMapper {
            name: "embedding",
            fieldType: KnnFloatVectorField.FieldType {
                dimension: 768,
                vectorEncoding: FLOAT32,
                similarityFunction: EUCLIDEAN  // DiskANN with L2
            },
            perDimensionValidator: FiniteValueValidator,
            vectorValidator: [
                DimensionValidator(768),
                SpaceVectorValidator(L2)  // No zero vector check
            ],
            vectorTransformer: IdentityTransformer,  // No transformation
            knnMethodContext: {
                engine: JVECTOR,
                spaceType: L2,
                methodComponentContext: {
                    name: "disk_ann",
                    parameters: {
                        m: 32,
                        ef_construction: 200,
                        alpha: 1.2,
                        beam_width: 128,
                        disk_mode: true  // DiskANN enabled
                    }
                }
            }
        }
    }
}
```