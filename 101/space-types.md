# Space Types

Space types define how vector similarity/distance is calculated. The project supports 7 space types defined in SpaceType.java:

1. L2 (Euclidean Distance) - DEFAULT
- Value: "l2"
- Use Cases: General-purpose similarity, geometric distance
- Supported Data Types: FLOAT, BYTE

2. COSINESIMIL (Cosine Similarity)
- Value: "cosinesimil"
- Use Cases: Text embeddings, document similarity, direction-based matching
- Supported Data Types: FLOAT, BYTE

3. INNER_PRODUCT (Dot Product)
- Value: "innerproduct"
- Use Cases: Recommendation systems, when vectors are normalized
- Supported Data Types: FLOAT, BYTE

4. L1 (Manhattan Distance)
- Value: "l1"
- Use Cases: Grid-based distances, sparse vectors
- Supported Data Types: FLOAT, BYTE

5. LINF (Chebyshev Distance)
- Value: "linf"
- Use Cases: Chess-board distance, maximum difference scenarios
- Supported Data Types: FLOAT, BYTE

6. HAMMING (Hamming Distance) - DEFAULT for BINARY
- Value: "hamming"
- Use Cases: Binary vectors, bit-level comparisons, error detection
- Supported Data Types: BINARY only