# Document Indexing

## 1. User Indexes Document

Add document with vector to index

**Input:**

```
POST /my-index/_doc/1
{
  "title": "Test Document",
  "embedding": [0.1, 0.2, 0.3, ..., 0.128]
}

```

## 2. Call Stack

Route document through parsing pipeline to field-specific mappers


```
RestIndexAction
  → TransportBulkAction.executeBulk()
    → InternalEngine.index()
      → DocumentParser.parseDocument()
        → LuceneFieldMapper.parse()
```

### 2.1. What Happens at Each Layer:

- RestIndexAction: HTTP → internal request object
- TransportBulkAction: Batching, routing to correct shard
- InternalEngine: Shard-level indexing coordination
- DocumentParser: JSON → Lucene Document conversion
- LuceneFieldMapper: Vector-specific parsing

## 3. Entry Point - parseCreateField()

Route vector field parsing based on data type (FLOAT, BYTE, BINARY)

**Implementation:**

```java
KNNVectorFieldMapper -> parseCreateField
```
**Outcome:**: Routes to getFloatsFromContext() for FLOAT vector

## 4. Parse Vector Array - getFloatsFromContext()
yyy
Parse JSON array token-by-token into primitive float[] with per-dimension validation

**Implementation:**

```java
KNNVectorFieldMapper -> getFloatsFromContext
```

**Outcome:**: Returns Optional.of(float[size]) with all dimensions validated

## 5. Vector Transformation

Apply transformations (normalization for COSINE, no-op for L2)

**Implementation:**

```java
VectorTransformer -> transform
```

**Outcome:**: Vector transformed (or unchanged for L2)

6.Check Derived Source Enabled

*Derived Source* is a storage optimization feature in OpenSearch k-NN plugin that dramatically reduces disk space usage for vector fields by storing a masked version in the _source field while keeping the full vector in other locations.

**Implementation:**

```java
KNNVectorFieldMapper -> isDerivedEnabled
```

**Outcome:**: Returns true - derived source enabled

## 6. Create Lucene Fields

Create all Lucene Field objects for this vector. It creates HNSW/disk-ann options with vector fields

**Implementation:**

```java
LuceneFieldMapper -> getFieldsForFloatVector
```

**Outcome:**: Returns List of Fields


## 7. Knn Vector Field

Create all Lucene Field objects for this vector. It creates HNSW/disk-ann options with vector fields

**Implementation:**

```java
new DerivedKnnFloatVectorField(name, fieldType, isDerivedEnabled)
```

**Outcome:**: Field created with isDerivedEnabled=true, it is important to have for engine side of indexing

## 8. Add Fields to Document

Add all created fields to the document
**Implementation:**

```java
context.doc().addAll(#7 fields);
```

**Outcome:**: Field created with isDerivedEnabled=true, it is important to have for engine side of indexing

### 9. Complete Outcome

```
ParseContext.Document {
    fields: [
        TextField("title", "Machine Learning Paper"),
        StoredField("_source", "{\"title\":\"...\",\"embedding\":[0.1,0.2,...,0.128]}"),
        DerivedKnnFloatVectorField("embedding", float[128], isDerivedEnabled=true),
        VectorField("embedding", BytesRef(512 bytes))
    ]
}
```
