```
opensearch-jvector/
├── src/main/java/org/opensearch/knn/
│   ├── plugin/
│   │   ├── JVectorKNNPlugin.java          # Main plugin entry point
│   │   ├── rest/                          # REST API handlers
│   │   ├── script/                        # Scripting support
│   │   ├── stats/                         # Statistics
│   │   └── transport/                     # Transport actions
│   │
│   ├── index/
│   │   ├── mapper/
│   │   │   ├── KNNVectorFieldMapper.java  # Field mapping
│   │   │   ├── MethodFieldMapper.java     # Method-based mapping
│   │   │   ├── LuceneFieldMapper.java     # Lucene integration
│   │   │   └── VectorTransformer.java     # Vector transformations
│   │   │
│   │   ├── codec/
│   │   │   ├── jvector/
│   │   │   │   ├── JVectorFormat.java     # Main codec format
│   │   │   │   ├── JVectorIndexWriter.java # Index writer
│   │   │   │   ├── JVectorReader.java     # Index reader
│   │   │   │   ├── JVectorKnnFloatVectorQuery.java
│   │   │   │   └── JVectorDiskANNMethod.java
│   │   │   │
│   │   │   ├── derivedsource/             # Derived source support
│   │   │   │   ├── DerivedSourceIndexOperationListener.java
│   │   │   │   ├── DerivedSourceStoredFieldsWriter.java
│   │   │   │   └── DerivedSourceStoredFieldsReader.java
│   │   │   │
│   │   │   ├── KNN80Codec/                # Legacy codec support
│   │   │   ├── KNN990Codec/               # Quantization support
│   │   │   └── util/                      # Codec utilities
│   │   │
│   │   ├── query/
│   │   │   ├── KNNQueryBuilder.java       # Query DSL builder
│   │   │   ├── KNNScorer.java             # Scoring logic
│   │   │   ├── RNNQueryFactory.java       # Query factory
│   │   │   ├── iterators/                 # Vector iterators
│   │   │   └── parser/                    # Query parsers
│   │   │
│   │   ├── engine/
│   │   │   ├── KNNEngine.java             # Engine abstraction
│   │   │   ├── JVMLibrary.java            # JVM library impl
│   │   │   ├── KNNMethod.java             # Method definitions
│   │   │   ├── MethodResolver.java        # Method resolution
│   │   │   └── lucene/                    # Lucene engine
│   │   │
│   │   ├── vectorvalues/                  # Vector value handling
│   │   ├── SpaceType.java                 # Distance metrics
│   │   ├── VectorDataType.java            # Data types
│   │   └── KNNSettings.java               # Configuration
│   │
│   ├── quantization/
│   │   ├── quantizer/
│   │   │   ├── OneBitScalarQuantizer.java
│   │   │   ├── MultiBitScalarQuantizer.java
│   │   │   └── BitPacker.java
│   │   ├── models/                        # Quantization models
│   │   └── factory/                       # Quantizer factory
│   │
│   └── common/
│       ├── KNNConstants.java              # Constants
│       ├── KNNValidationUtil.java         # Validation
│       └── exception/                     # Custom exceptions
│
├── benchmark-jmh/                         # JMH benchmarks
├── qa/                                    # Quality assurance tests
├── docs/                                  # Documentation
└── build.gradle                           # Build configuration
```