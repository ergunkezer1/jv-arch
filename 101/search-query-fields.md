## Search Query Field Descriptions

### Basic Search

```json
{
  "size": 10,                    // Number of results to return
  "query": {
    "knn": {
      "embedding": {             // Vector field name
        "vector": [...],         // Query vector (must match dimension)
        "k": 10                  // Number of nearest neighbors to find
      }
    }
  }
}
```

### Query Field Descriptions

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `size` | integer | No | Number of results to return (default: 10, max: 10,000) |
| `vector` | array | Yes | Query vector - must have same dimension as indexed vectors |
| `k` | integer | Yes | Number of nearest neighbors to find (max: 10,000) |
| `filter` | object | No | Standard OpenSearch filter to restrict candidates |
| `method_parameters` | object | No | Query-time tuning parameters (engine-specific) |
