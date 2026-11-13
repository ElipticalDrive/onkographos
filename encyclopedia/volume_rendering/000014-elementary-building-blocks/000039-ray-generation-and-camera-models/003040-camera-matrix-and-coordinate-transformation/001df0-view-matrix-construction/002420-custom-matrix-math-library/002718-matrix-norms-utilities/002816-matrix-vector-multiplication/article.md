# Matrix-Vector Multiplication
Also called: operator* (Mat4, Vec4)
## Overview
Computes the product of a 4x4 matrix and a 4D vector, yielding a transformed 4D vector [S1]. It performs a linear combination of the matrix columns weighted by the vector components [G]. Unlike symmetric matrix structures that exploit redundancy, this method applies a full 4×4 transformation to any 4D vector [G].
## How it works
**Inputs:**
- `m`: A 4x4 matrix [S1].
- `v`: A 4D vector [S1].
**Outputs:**
- `result`: A 4D vector representing $m \times v$ [S1].
**Data Structures:**
The exact memory layout and bit packing of the matrix and vector are not given in the sources [S1]. Standard implementations store the matrix as a 4x4 grid of elements $m_{ij}$ and the vector as a 4-element array $v_j$ [G].
**Steps:**
1. Initialize a 4D result vector `r`.
2. For each row index $i$ from 0 to 3:
3. Compute the dot product of the $i$-th row of `m` and the vector `v` [G].
4. Store the result in `r[i]`.
5. Return `r`.
**Formula:**
The result vector $r$ is computed as:
$$ r_i = \sum_{j=0}^{3} m_{ij} v_j $$
where $i \in \{0, 1, 2, 3\}$ is the row index, $j \in \{0, 1, 2, 3\}$ is the column/component index, $m_{ij}$ is the element at row $i$ and column $j$ of the matrix, and $v_j$ is the $j$-th component of the vector [G].
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def matrix_vector_multiply(m, v):
    r = [0.0, 0.0, 0.0, 0.0]
    for i in range(4):
        acc = 0.0
        for j in range(4):
            acc += m[i][j] * v[j]
        r[i] = acc
    return r
```
**Parameters:**
- `T`: The numeric type (e.g., `float`, `double`) used in the template signature [S1]. Not given in the sources.
**Edge Cases and Pitfalls:**
- Precision: Accumulation of floating-point errors in the summation may occur [G].
- Homogeneous Coordinates: If `v` represents a point, the resulting `w` component (index 3) may not be 1.0, requiring perspective division in subsequent steps [G].
- Degenerate Input: If the matrix is singular, the transformation may collapse dimensions [G].
## Subtopics

### [Scalar-Vector Multiplication](6274-scalar-vector-multiplication.md)
Multiplies a scalar by a 4-dimensional vector. It uniformly scales the vector's magnitude without altering its direction [G]. Choose it when only uniform scaling is required, as it avoids the overhead of a full matrix transformation [G].

## Operating characteristics
- Time Complexity: $O(1)$ for fixed 4x4 matrix and 4D vector, involving 16 multiplications and 12 additions [G].
- Memory Complexity: $O(1)$ auxiliary space [G].
- Measured Performance: Not given in the sources [S1].
- Scaling: Linear with respect to vector dimension if generalized, but fixed for 4D [G].
- Strengths: Efficient, hardware-accelerable, fundamental for transformation pipelines [G].
- Limitations: Requires consistent data types; precision issues may arise in specific numerical regimes [G].
- When to Use: Applying camera transformations, object transformations in volume rendering [G].
- When Not to Use: When only scaling is needed (use scalar-vector multiplication) [G].
## Background
Originated in the OpenVDB library's `nanovdb::math` namespace, specifically as an overloaded `operator*` for `Mat4` and `Vec4` types [S1]. Not given in the sources.
## Sources
[S1] OpenVDB: nanovdb::math Namespace Reference, https://openvdb.org/documentation/doxygen/namespacenanovdb_1_1math.html, fetched 2026-10-24
