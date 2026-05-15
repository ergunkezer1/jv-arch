# Vector Data Types

The project supports 3 vector data types defined in VectorDataType.java:

1. FLOAT (Default)
- Description: Standard floating-point vectors
- Supported by: Both Lucene and JVector engines

2. BYTE
- Description: 8-bit integer vectors (values -128 to 127)
- Supported by: Lucene engine only

3. BINARY
- Description: Binary vectors (packed bits)
- Supported by: Lucene engine only
- Note: Uses custom BinaryVectorScorer