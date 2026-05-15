# Vector Parsing

Parse incoming document, extract vector field, validate dimensions and values, apply transformations, create Lucene fields for codec processing.

## Call Stack

```
REST API: PUT /my-index/_doc/1
  └─> RestIndexAction.prepareRequest()
      └─> TransportIndexAction.doExecute()
          └─> InternalEngine.index()
              └─> DocumentParser.parseDocument()
                  └─> DocumentMapper.parse()
                      └─> KNNVectorFieldMapper.parseCreateField()
                          ├─> Parse JSON array → float[768]
                          ├─> Validation chain:
                          │   ├─> FiniteValueValidator.validate() (both modes)
                          │   ├─> DimensionValidator.validate() (both modes)
                          │   └─> SpaceTypeValidator.validate()
                          │       ├─> HNSW: Check not zero vector ✓
                          │       └─> DiskANN: L2-specific validation ✓
                          ├─> Transformation:
                          │   ├─> HNSW: NormalizeVectorTransformer.transform()
                          │   │   ├─> magnitude
                          │   │   └─> v[i] = v[i] / magnitude
                          │   │       Normalize to unit length for COSINE
                          │   └─> DiskANN: IdentityTransformer.transform()
                          │       └─> return vector unchanged
                          │           No normalization for L2
                          └─> Create Lucene fields:
                              ├─> DerivedKnnFloatVectorField (both modes)
                              ├─> StoredField (both modes)
                              └─> TextField (both modes)
```

## Key Files

- KNNVectorFieldMapper.java - Entry point
- NormalizeVectorTransformer.java - HNSW normalization
- DerivedKnnFloatVectorField.java - Lucene field wrapper

## Purpose

Convert user JSON vector into validated, transformed Lucene fields ready for codec processing.

## What Happens?

1. User indexes document: PUT /my-index/_doc/1 with embedding array
2. KNNVectorFieldMapper.parseCreateField() invoked
3. Parse JSON array → float[768]
4. Validation (both modes):
    - FiniteValueValidator
    - DimensionValidator
    - HNSW: SpaceTypeValidator ensures not zero vector
    - DiskANN: SpaceVectorValidator L2-specific validation
5. Transformation:
    - HNSW: L2 normalize: magnitude=1.7320508, v[i]=v[i]/magnitude
    - DiskANN: Identity: return vector unchanged
6. Create 3 Lucene fields (both modes):
    - DerivedKnnFloatVectorField: Primary vector
    - StoredField: Compressed original for _source
    - TextField: Other document fields
7. Add fields to Lucene document

## Outcome

- Vector validated against all constraints
- HNSW: Vector normalized to unit length
- DiskANN: Vector kept in original form
- Three Lucene fields created
- Document ready for codec layer

## Status - HNSW

```java
ParsedDocument {
    docId: 1,
    luceneDocument: Document {
        fields: [
            DerivedKnnFloatVectorField {
                name: "embedding",
                vectorValue: float[768] {
                    [0.0577, 0.1155, 0.1732, ..., 0.5196]  // Normalized
                },
                fieldType: {
                    dimension: 768,
                    similarityFunction: COSINE
                }
            },
            StoredField {name: "embedding", binaryValue: byte[3072]},
            TextField {name: "title", value: "Example document"}
        ]
    },
    vectorState: {
        originalVector: [0.1, 0.2, 0.3, ..., 0.9],
        magnitude: 1.7320508,
        normalizedVector: [0.0577, 0.1155, ..., 0.5196],
        transformationApplied: "L2_NORMALIZATION"
    }
}
```

## Status - DiskANN

```java
ParsedDocument {
    docId: 1,
    luceneDocument: Document {
        fields: [
            DerivedKnnFloatVectorField {
                name: "embedding",
                vectorValue: float[768] {
                    [0.1, 0.2, 0.3, ..., 0.9]  // Original values
                },
                fieldType: {
                    dimension: 768,
                    similarityFunction: EUCLIDEAN
                }
            },
            StoredField {name: "embedding", binaryValue: byte[3072]},
            TextField {name: "title", value: "Example document"}
        ]
    },
    vectorState: {
        originalVector: [0.1, 0.2, 0.3, ..., 0.9],
        transformedVector: [0.1, 0.2, 0.3, ..., 0.9],  // Same
        transformationApplied: "IDENTITY"
    }
}
```

