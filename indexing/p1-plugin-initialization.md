# Plugin Initialization

Bootstrap the k-NN plugin during OpenSearch cluster startup, registering custom field types and codec implementations.

## Call Stack

```
OpenSearch Cluster Startup
  └─> PluginsService.loadPlugins()
      └─> JVectorKNNPlugin.<init>()
          ├─> JVectorKNNPlugin.getMappers()
          │   └─> Returns: {"knn_vector" → KNNVectorFieldMapper.TypeParser}
          │       KNNVectorFieldMapper.TypeParser - Parses knn_vector field definitions from index mappings
          ├─> JVectorKNNPlugin.getCodecs()
          │   └─> Returns: {"KNN1030Codec", "KNN10010Codec"}
          │       KNN1030Codec - Legacy codec for backward compatibility
          │       KNN10010Codec - Current codec with derived source support
          └─> JVectorKNNPlugin.onIndexModule()
              └─> Registers index operation listeners
                   DerivedSourceIndexOperationListener - Tracks derived source operations

```


## Key Files

- JVectorKNNPlugin.java:87-100 - Main plugin class, registers mappers and codecs
- KNNVectorFieldMapper.java:326 - TypeParser for knn_vector field type
- KNN10010Codec.java - Current codec implementation

## Purpose

Make the knn_vector field type available and register JVector codec for both HNSW and DiskANN modes.

## What Happens

1. OpenSearch PluginsService discovers and loads JVectorKNNPlugin from classpath
2. Plugin's getMappers() registers "knn_vector" → KNNVectorFieldMapper.TypeParser
3. Plugin's getCodecs() registers codec implementations for vector storage
4. Codec services registered via META-INF/services/org.apache.lucene.codecs.Codec
5. Thread pools initialized for SIMD operations
6. Index operation listeners registered for lifecycle hooks


## Outcome

- knn_vector field type becomes available for index mappings
- JVector codec registered and ready to handle vector fields
- Plugin infrastructure initialized and operational
- Cluster can now accept k-NN index creation requests

## Status

```java
ClusterState {
    plugins: [
        JVectorKNNPlugin {
            name: "opensearch-knn",
            version: "2.18.0",
            mappers: {
                "knn_vector": KNNVectorFieldMapper.TypeParser
            },
            codecs: {
                "KNN1030Codec": KNN1030Codec,
                "KNN10010Codec": KNN10010Codec
            },
            listeners: [
                DerivedSourceIndexOperationListener
            ],
            initialized: true
        }
    ],
    availableFieldTypes: ["text", "keyword", "long", "knn_vector", ...]
}

```