### Plugin Layer (JVectorKNNPlugin.java)
- Entry point for the plugin
- Registers mappers, queries, codecs, and REST handlers
- Implements multiple OpenSearch plugin interfaces

### Mapper Layer (KNNVectorFieldMapper.java)
- Defines knn_vector field type
- Handles vector field parsing and validation
- Supports dimension, space type, and method configuration

### Query Layer (KNNQueryBuilder.java)
- Builds KNN queries from DSL
- Supports k-NN, radius search, and filtered search
- Integrates with OpenSearch query framework

### Codec Layer (JVectorFormat.java)
- Custom Lucene codec for vector storage
- Implements incremental merges
- Supports quantization and DiskANN

### Engine Layer (JVMLibrary.java)
- Abstraction over JVector library
- Manages HNSW(Hierarchical Navigable Small World) graph construction
- Handles method resolution and configuration

### Quantization Layer (quantization/)
- Scalar quantization (1-bit, multi-bit)
- Product quantization (PQ)
- Quantization state caching