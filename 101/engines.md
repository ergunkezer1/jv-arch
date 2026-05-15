## Engine 1: JVector (DISK_ANN) - DEFAULT

**Method:** "disk_ann" (Disk-based Approximate Nearest Neighbor)

**Supported:**
- Data Types: FLOAT only
- Space Types: L2, L1, LINF, COSINESIMIL, INNER_PRODUCT

**Parameters:**

- m: Max connections per node (default: 16)
- ef_construction: Construction-time search depth (default: 100)
- advanced.alpha: Diversity parameter (default: 1.2)
- advanced.neighbor_overflow: Overflow factor (default: 1.2)
- advanced.hierarchy_enabled: Enable hierarchical structure (default: false)
- advanced.min_batch_size_for_quantization: Min vectors for quantization (default: 1024)
- advanced.num_pq_subspaces: Product quantization subspaces (optional)
- advanced.leading_segment_merge_disabled: Disable leading segment merges (default: false)

**Query Parameters:**

- ef_search: Search-time candidate list size
- overquery_factor: Reranking multiplier (default: 5)
- advanced.threshold: Similarity threshold (default: 0.0)
- advanced.rerank_floor: Minimum rerank threshold (default: 0.0)
- advanced.use_pruning: Enable pruning optimization (default: false)

### Example 1. Index Creation Minimal

```bash
curl -X PUT "http://localhost:9200/jvector-index" \
  -H "Content-Type: application/json" -d '{
  "settings": {
    "index.knn": true
  },
  "mappings": {
    "properties": {
      "embedding": {
        "type": "knn_vector",
        "dimension": 128,
        "method": {
          "name": "disk_ann",
          "engine": "jvector",
          "space_type": "l2"
        }
      }
    }
  }
}'
```

### Example 2. Index Creation Full Configuration

```bash
curl -X PUT "http://localhost:9200/jvector-advanced" \
  -H "Content-Type: application/json" -d '{
  "settings": {
    "index.knn": true,
    "number_of_shards": 1,
    "number_of_replicas": 0
  },
  "mappings": {
    "properties": {
      "embedding": {
        "type": "knn_vector",
        "dimension": 128,
        "method": {
          "name": "disk_ann",
          "engine": "jvector",
          "space_type": "cosinesimil",
          "parameters": {
            "m": 16,
            "ef_construction": 100,
            "advanced.alpha": 1.2,
            "advanced.neighbor_overflow": 1.2,
            "advanced.hierarchy_enabled": false,
            "advanced.min_batch_size_for_quantization": 1024,
            "advanced.num_pq_subspaces": 16
          }
        }
      }
    }
  }
}'
```

### Example 3. Search Query Basic

```bash
curl -X POST "http://localhost:9200/jvector-index/_search" \
  -H "Content-Type: application/json" -d '{
  "size": 5,
  "query": {
    "knn": {
      "embedding": {
        "vector": [0.15, 0.25, 0.35, ..., 0.128],
        "k": 5
      }
    }
  }
}'
```

### Example 4. Search Query Advanced

```bash
curl -X POST "http://localhost:9200/jvector-index/_search" \
  -H "Content-Type: application/json" -d '{
  "size": 10,
  "query": {
    "knn": {
      "embedding": {
        "vector": [0.15, 0.25, 0.35, ..., 0.128],
        "k": 10,
        "method_parameters": {
          "ef_search": 100,
          "overquery_factor": 5,
          "advanced.threshold": 0.0,
          "advanced.rerank_floor": 0.0
        },
        "filter": {
          "term": { "category": "technology" }
        }
      }
    }
  }
}'
```

## Engine 2: Lucene (HNSW)

Method: "hnsw" (Hierarchical Navigable Small World)

**Supported:**
- Data Types: FLOAT, BYTE, BINARY
- Space Types: L2, COSINESIMIL, INNER_PRODUCT, HAMMING

**Parameters:**
- m: Max connections per node (default: 16)
- ef_construction: Construction-time search depth (default: 100)
- encoder: Optional scalar quantization

### Example 1. Index Creation

```bash
curl -X PUT "http://localhost:9200/lucene-index" \
  -H "Content-Type: application/json" -d '{
  "settings": {
    "index.knn": true
  },
  "mappings": {
    "properties": {
      "my_vector": {
        "type": "knn_vector",
        "dimension": 128,
        "method": {
          "name": "hnsw",
          "engine": "lucene",
          "space_type": "l2",
          "parameters": {
            "m": 16,
            "ef_construction": 100
          }
        }
      }
    }
  }
}'
```

### Example 2. Index Documents

```bash
curl -X POST "http://localhost:9200/lucene-index/_doc/1" \
  -H "Content-Type: application/json" -d '{
  "my_vector": [0.1, 0.2, 0.3, ..., 0.128],
  "title": "Document 1"
}'
```

### Example 3. Search Query

```bash
curl -X POST "http://localhost:9200/lucene-index/_search" \
  -H "Content-Type: application/json" -d '{
  "size": 5,
  "query": {
    "knn": {
      "my_vector": {
        "vector": [0.15, 0.25, 0.35, ..., 0.128],
        "k": 5
      }
    }
  }
}'
```