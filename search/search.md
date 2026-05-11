## Search Query Execution Flow

### Example Query

```json
POST /products/_search
{
  "query": {
    "knn": {
      "title_embedding": {
        "vector": [0.1, 0.2, ..., 0.768],
        "k": 10,
        "method_parameters": {
          "overquery_factor": 5
        }
      }
    }
  }
}
```

### Step-by-Step Execution

#### **Step 1: Query Parsing & Validation**

**File:** `KNNQueryBuilder.java:376-411`

**Query State:**
```
Input: JSON query
↓
Parsed State:
  - fieldName: "title_embedding"
  - vector: [0.1, 0.2, ..., 0.768] (768 dimensions)
  - k: 10
  - methodParameters: {overquery_factor: 5}
```

**Key Operations:**
```java
// Get field mapping
MappedFieldType mappedFieldType = context.fieldMapper(this.fieldName);
KNNVectorFieldType knnVectorFieldType = (KNNVectorFieldType) mappedFieldType;

// Extract configuration from mapping
KNNEngine knnEngine = queryConfigFromMapping.get().getKnnEngine();  // JVECTOR
SpaceType spaceType = queryConfigFromMapping.get().getSpaceType();  // L2, COSINE, etc.
VectorDataType vectorDataType = queryConfigFromMapping.get().getVectorDataType();  // FLOAT

// Transform vector if needed (e.g., normalize for cosine)
knnVectorFieldType.transformQueryVector(vector);
```

**Query State After Step 1:**
```
Validated Query Object:
  - field: "title_embedding"
  - vector: [0.1, 0.2, ..., 0.768] (normalized if cosine)
  - k: 10
  - engine: JVECTOR
  - spaceType: COSINESIMIL
  - vectorDataType: FLOAT
  - overQueryFactor: 5
  - threshold: 0.0
  - rerankFloor: 1.0
  - usePruning: false
```

---

#### **Step 2: Query Object Creation**

**File:** `KNNQueryBuilder.java:552-589`

**Key Operation:**
```java
return new JVectorKnnFloatVectorQuery(
    this.fieldName,           // "title_embedding"
    target,                   // [0.1, 0.2, ..., 0.768]
    k,                        // 10
    filterQuery,              // MatchAllDocsQuery or actual filter
    overQueryFactor,          // 5
    threshold,                // 0.0
    rerankFloor,              // 1.0
    usePruning                // false
);
```

**Query State:**
```
JVectorKnnFloatVectorQuery Object:
  ├─ field: "title_embedding"
  ├─ target: float[768]
  ├─ k: 10
  ├─ filter: Query object
  ├─ overQueryFactor: 5
  ├─ threshold: 0.0
  ├─ rerankFloor: 1.0
  └─ usePruning: false
```

**Next:** Query object returned to Lucene's IndexSearcher

---

#### **Step 3: Per-Segment Search**

**File:** `JVectorKnnFloatVectorQuery.java:62-82`

**Caller:** Lucene's `KnnFloatVectorQuery` parent class (per segment)

**Query State at Entry:**
```
For Segment 1 (500K docs):
  - LeafReaderContext: segment1
  - Query params: same as above
  - AcceptDocs: filter bitmap (or null)
```

**Key Operations:**
```java
// Create JVector-specific collector
final KnnCollector knnCollector = new JVectorKnnCollector(
    delegateCollector, 
    threshold,          // 0.0
    rerankFloor,        // 1.0
    overQueryFactor,    // 5
    usePruning          // false
);

// Delegate to codec's reader
reader.searchNearestVectors(field, getTargetCopy(), knnCollector, acceptDocs);
```

**Query State:**
```
Per-Segment Execution:
  - Segment: 1 of 3
  - Field: "title_embedding"
  - Target vector: float[768]
  - Collector: JVectorKnnCollector(k=10, overQueryFactor=5)
  - Filter: AcceptDocs bitmap
```

---

#### **Step 4: Graph Loading**

**File:** `JVectorReader.java:132-167`

**Query State:**
```
Loaded Resources for "title_embedding":
  ├─ OnDiskGraphIndex: HNSW graph structure
  ├─ PQVectors: Compressed vectors (if Quantisation enabled)
  ├─ GraphNodeIdToDocMap: node_id => doc_id mapping
  └─ VectorSimilarityFunction: Distance metric (COSINE)
```

**Key Operations:**
```java
// Get graph for this field
final OnDiskGraphIndex index = fieldEntryMap.get(field).index;

// Check for Product Quantisation
if (fieldEntryMap.get(field).pqVectors != null) {
    // Two-phase scoring: PQ (fast) + exact (accurate)
    ScoreFunction.ApproximateScoreFunction asf = pqVectors.precomputedScoreFunctionFor(q, similarityFunction);
    ScoreFunction.ExactScoreFunction reranker = view.rerankerFor(q, similarityFunction);
    ssp = new DefaultSearchScoreProvider(asf, reranker);
} else {
    // Single-phase: full-precision only
    ssp = DefaultSearchScoreProvider.exact(q, similarityFunction, view);
}
```

---

#### **Step 5: Graph Traversal**

**File:** `JVectorReader.java:180-188`

**Query State:**
```
Graph Search Parameters:
  - k: 10 (desired results)
  - candidates: 50 (k × overQueryFactor = 10 × 5)
  - threshold: 0.0 (minimum similarity)
  - rerankFloor: 1.0 (re-rank multiplier)
  - filter: Lucene doc filter
```

**Key Operation:**
```java
final var searchResults = graphSearcher.search(
    ssp,                                                          // Score provider
    jvectorKnnCollector.k(),                                      // 10
    jvectorKnnCollector.k() * jvectorKnnCollector.getOverQueryFactor(),  // 50
    jvectorKnnCollector.getThreshold(),                           // 0.0
    jvectorKnnCollector.getRerankFloor(),                         // 1.0
    compatibleBits                                                // Filter
);
```

**Graph Traversal Process:**
```
1. Start at entry point (e.g., node 42)
2. Score neighbors with PQ vectors (fast):
   node 43: 0.85, node 44: 0.82, node 45: 0.79
3. Move to best neighbor (node 43)
4. Repeat until 50 candidates collected
5. Apply filter (skip filtered docs)
```

**Query State After Traversal:**
```
Candidates Found: 50 nodes
  - Scored with PQ (approximate)
  - Filtered by AcceptDocs
  - Example: [node47: 0.88, node43: 0.85, ..., node99: 0.72]
```

---

#### **Step 6: Re-ranking**

**Happens inside:** `graphSearcher.search()` (JVector library)

**Query State:**
```
Re-ranking Phase:
  Input: 50 candidates with PQ scores
  ↓
  Load full-precision vectors for each candidate
  ↓
  Re-score with exact distance calculation
  ↓
  Output: 50 candidates with exact scores
```

**Example:**
```
Before re-ranking (PQ scores):
  [node47: 0.88, node43: 0.85, node52: 0.84, ..., node99: 0.72]

After re-ranking (exact scores):
  [node52: 0.91, node47: 0.89, node43: 0.87, ..., node99: 0.70]
```

---

#### **Step 7: Top-K Selection**

**File:** `JVectorReader.java:189-191`

**Query State:**
```
Selection Process:
  Input: 50 re-ranked candidates
  ↓
  Select top 10 by score
  ↓
  Map node IDs => doc IDs
  ↓
  Output: Top 10 documents
```

**Key Operation:**
```java
for (SearchResult.NodeScore ns : searchResults.getNodes()) {
    // Map jVector node ID to Lucene doc ID
    int docId = jvectorLuceneDocMap.getLuceneDocId(ns.node);
    jvectorKnnCollector.collect(docId, ns.score);
}
```

**Query State After Step 7:**
```
Segment 1 Results:
  [(doc1523, 0.91), (doc892, 0.89), (doc2341, 0.87), ..., (doc445, 0.82)]
  Total: 10 documents
```

---

#### **Step 8: Statistics Collection**

**File:** `JVectorReader.java:197-207`

**Query State:**
```
Search Metrics (Segment 1):
  - visitedNodesCount: 150
  - rerankedCount: 50
  - expandedCount: 200
  - expandedBaseLayerCount: 180
  - searchTime: 5ms
```

**Storage:**
```java
KNNCounter.KNN_QUERY_VISITED_NODES.add(visitedNodesCount);
KNNCounter.KNN_QUERY_RERANKED_COUNT.add(rerankedCount);
KNNCounter.KNN_QUERY_EXPANDED_NODES.add(expandedCount);
KNNCounter.KNN_QUERY_GRAPH_SEARCH_TIME.add(searchTime);
```

**Access:** `GET /_nodes/stats/indices/knn`

---

#### **Step 9: Multi-Segment Merge**

**Performed by:** Lucene's IndexSearcher (not in opensearch-jvector code)

**Query State:**
```
Segment Results:
  Segment 1 (500K docs): 10 results, best score: 0.91
  Segment 2 (300K docs): 10 results, best score: 0.88
  Segment 3 (200K docs): 10 results, best score: 0.85
  ↓
  Merge all 30 results
  ↓
  Sort by score globally
  ↓
  Select top 10
```

**Final Query State:**
```
Global Top 10:
  [
    (doc1523, 0.91),  // from segment 1
    (doc892, 0.89),   // from segment 1
    (doc2001, 0.88),  // from segment 2
    (doc2341, 0.87),  // from segment 1
    ...
  ]
```

---

## Architecture Components

### 1. Codec System

**Purpose:** Pluggable format for reading/writing vector data

**Key Classes:**
- `JVectorFormat`: Defines format name and creates readers/writers
- `JVectorWriter`: Writes graphs during indexing/merging
- `JVectorReader`: Reads graphs during search
- `BasePerFieldKnnVectorsFormat`: Routes fields to appropriate formats

**Source:** `JVectorFormat.java:186-189`
```java
@Override
public KnnVectorsReader fieldsReader(SegmentReadState state) throws IOException {
    return new JVectorReader(state);  // One reader per segment
}
```

### 2. Graph Structure

**Storage:** On-disk with memory-mapped access

**Components:**
- **Nodes:** Vector ordinals (not doc IDs)
- **Edges:** Neighbor connections
- **Layers:** Hierarchical structure for faster search
- **Mapping:** `GraphNodeIdToDocMap` translates node IDs => doc IDs

**Why separate node IDs?** Doc IDs change during merges; node IDs remain stable

### 3. Product Quantisation

**Purpose:** Compress vectors for memory efficiency

**Process:**
1. **Training:** Learn codebooks from sample vectors
2. **Encoding:** Replace vector chunks with codebook indices
3. **Search:** Use compressed vectors for fast approximate scoring
4. **Re-ranking:** Use full vectors for accurate final scores

**Compression ratio:** 768 dims × 4 bytes = 3,072 bytes => 192 bytes (16x smaller)

**Source:** `JVectorWriter.java:392`
```java
ProductQuantisation pq = ProductQuantisation.compute(
    randomAccessVectorValues,
    M,  // 192 subspaces for 768-dim vectors
    numberOfClustersPerSubspace,  // 256 clusters per subspace
    ...
);
```

### 4. Incremental Merges

**Traditional HNSW:** Rebuild entire graph when merging segments

**JVector optimization:** Extend largest segment's graph with new vectors

**Source:** `JVectorWriter.java:899`
```java
// In the event of no deletes or Quantisation, the graph construction 
// is done by incrementally adding vectors from smaller segments 
// into the largest segment.
```

**Benefit:** Faster merges, especially for large segments

---

## Code Execution Hierarchy

### Complete Call Stack

```
1. User Query
   POST /products/_search

2. KNNQueryBuilder.doToQuery() [Line 376]
   ├─ Validate field type
   ├─ Get engine from mapping (JVECTOR)
   ├─ Transform vector (normalize if cosine)
   └─ Create JVectorKnnFloatVectorQuery [Line 580]

3. Lucene IndexSearcher.search()
   └─ For each segment (parallel)

4. JVectorKnnFloatVectorQuery.approximateSearch() [Line 62]
   ├─ Create JVectorKnnCollector
   └─ Call reader.searchNearestVectors() [Line 79]

5. JVectorReader.search() [Line 132]
   ├─ Load graph for field [Line 133]
   ├─ Check PQ compression [Line 155]
   ├─ Create score provider [Line 159-166]
   └─ Execute graph search [Line 181]

6. GraphSearcher.search() (JVector library)
   ├─ Traverse graph (~150 nodes visited)
   ├─ Score with PQ (50 candidates)
   ├─ Re-rank with full vectors
   └─ Return top 10

7. Map node IDs => doc IDs [Line 189-191]

8. Record statistics [Line 197-207]

9. Return segment results [Line 80]

10. Lucene merges all segments => Global top 10

11. Return JSON response to user
```

---

## Performance

### 1. Two-Phase Search

**Phase 1:** Fast approximate scoring with PQ vectors
- Check 50 candidates (k × overQueryFactor)
- Use compressed vectors (16x smaller)
- ~10x faster than full-precision

**Phase 2:** Accurate re-ranking with full vectors
- Re-score only 50 candidates (not all vectors)
- Use full-precision vectors
- Ensures high recall

### 2. Overquery Factor

**Purpose:** Improve recall by checking more candidates

**Example:** k=10, overQueryFactor=5
- Find 50 candidates with PQ
- Re-rank all 50
- Return best 10

**Trade-off:** Higher overQueryFactor = better recall, slower search

### 3. Graph Pruning

**Parameter:** `usePruning` (default: false)

**Purpose:** Skip nodes that can't be in top-k

**Example:** If current top-10 all have score > 0.9, skip nodes with score < 0.9

---

## Additional conceptual info:

### 1. Per-Field Independence

**Each vector field is completely independent:**
- Separate graph
- Separate PQ codebooks
- Separate node-to-doc mapping

**Example:** Query on `title_embedding` never touches `description_embedding`

### 2. Query Parameters vs Index Parameters

**Index-time (in mapping):**
- `ef_construction`: Graph quality during indexing
- `m`: Max neighbors per node
- `alpha`: Diversity factor

**Query-time (in search request):**
- `overquery_factor`: Candidate multiplier
- `threshold`: Minimum similarity
- `usePruning`: Enable/disable pruning

### 3. Vector Transformation

**Cosine similarity requires normalization:**

**Source:** `NormalizeVectorTransformer.java:18`
```java
VectorUtil.l2normalize(vector);  // Makes magnitude = 1
```

**Why?** Cosine only cares about direction, not magnitude

### 4. Statistics Monitoring

**Available metrics:**
- `knn_query_visited_nodes`: Nodes checked during search
- `knn_query_reranked_count`: Vectors re-ranked with full precision
- `knn_query_expanded_nodes`: Neighbors explored
- `knn_query_graph_search_time`: Time spent in graph traversal

**Access:** `GET /_nodes/stats/indices/knn`

### 5. Memory vs Accuracy Trade-offs

Based on above, we can see that:

| Configuration | Memory | Speed | Accuracy |
|---------------|--------|-------|----------|
| No PQ, low overQueryFactor | High | Fast | Medium |
| No PQ, high overQueryFactor | High | Slow | High |
| PQ, low overQueryFactor | Low | Very Fast | Low |
| PQ, high overQueryFactor | Low | Medium | High |

### 6. Distance Metrics

**Source:** `SpaceType.java:29-100`

- **L2 (Euclidean)**: `sqrt(Σ(a[i]-b[i])²)` - Geometric distance
- **Cosine**: `1 - (a·b)/(||a||×||b||)` - Directional similarity
- **Inner Product**: `a·b` - Dot product, magnitude matters

### 7. Key Parameters

| Parameter | Default | Purpose | Source |
|-----------|---------|---------|--------|
| `ef_construction` | 100 | Candidates checked during indexing | `JVectorDiskANNMethod.java:45` |
| `m` (max_connections) | 32 | Max neighbors per node | `JVectorFormat.java:34` |
| `alpha` | 1.2 | Diversity factor | `KNNConstants.DEFAULT_ALPHA_VALUE` |
| `neighbor_overflow` | 1.2 | Temporary neighbor excess | `KNNConstants.DEFAULT_NEIGHBOR_OVERFLOW_VALUE` |
| `overquery_factor` | 5 | Candidate multiplier at search | `KNNConstants.DEFAULT_OVER_QUERY_FACTOR` |

---

## Dictionary/Glossary

- **ANN**: Approximate Nearest Neighbor
- **HNSW**: Hierarchical Navigable Small World (graph algorithm)
- **PQ**: Product Quantisation (vector compression)
- **Codec**: coder-decoder defines how Lucene reads/writes index data to disk. It's a pluggable format specification.
- **Weight creation:** An object that holds query execution state and scoring logic
- **Segment**: Independent mini-index in Lucene
- **Node ID**: JVector's internal identifier (stable across merges)
- **Overquery Factor**: During graph traversal, keep more candidates than needed, then select best k. For eg: k = 10, with over query factor set to 5, we multiply k with 5 and keep 50 candidates for lookup. Then we rerank all and send only top 10. This gives us better lookup and precision
- **Re-ranking**: Scoring candidates with full-precision vectors
- **Incremental merges**: When Lucene combines multiple index segments, traditional HNSW rebuilds the entire graph from scratch. JVector extends the existing graph by adding new nodes.
- **Full precision vectors:** original uncompressed vectors
- **Graph Pruning:** Skip nodes that can't be in top-k results while searching
- **Magnitude**: Length of a vector (similar to the dimensions but dimension is how many elements are in a vector so bascially num of elements in a vector). Magnitude is basically how far the vector is from origin. For: vector [3, 4] in 2D space: dimensions is 2 but magnitude(distance from origin) is 5. Formula for magnitude: `sqrt(n1^2 + n2^2 + .... + nx^2)`;