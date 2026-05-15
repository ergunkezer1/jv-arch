# Index Field Descriptions

## Fields

```json
{
  "settings": {
    "index.knn": true,              // REQUIRED: Enables k-NN functionality
    "number_of_shards": 1,          // Number of primary shards (default: 1)
    "number_of_replicas": 0         // Number of replica shards (default: 0)
  },
  "mappings": {
    "properties": {
      "my_vector": {
        "type": "knn_vector",       // REQUIRED: Declares this as a vector field
        "dimension": 128,            // REQUIRED: Vector dimension (must match your embeddings)
        "method": {
          "name": "...",             // REQUIRED: Algorithm name ("hnsw" or "disk_ann")
          "engine": "...",           // REQUIRED: Engine name ("lucene" or "jvector")
          "space_type": "...",       // REQUIRED: Distance metric (see below)
          "parameters": { }          // OPTIONAL: Algorithm-specific parameters
        }
      }
    }
  }
}
```

## Descriptions

### Settings Section

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `index.knn` | boolean |  Yes | Must be `true` to enable k-NN plugin functionality on this index |
| `number_of_shards` | integer |  No | Number of primary shards (default: 1). More shards = better parallelism but more overhead |
| `number_of_replicas` | integer |  No | Number of replica copies (default: 0). Replicas provide fault tolerance |

### Mappings Section

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | Must be `"knn_vector"` for vector fields |
| `dimension` | integer | Yes | Vector dimension (1-16,000). Must exactly match your embedding model's output |
| `method.name` | string | Yes | Algorithm: `"hnsw"` (Lucene) or `"disk_ann"` (JVector) |
| `method.engine` | string | Yes | Engine: `"lucene"` or `"jvector"` |
| `method.space_type` | string | Yes | Distance metric (see Space Types below) |
| `method.parameters` | object | ❌ No | Algorithm-specific tuning parameters |


---

### Index Creation Example (Full Configuration)

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
        "dimension": 768,
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
            "advanced.num_pq_subspaces": 16,
            "advanced.leading_segment_merge_disabled": false
          }
        }
      },
      "title": { "type": "text" },
      "category": { "type": "keyword" }
    }
  }
}'
```