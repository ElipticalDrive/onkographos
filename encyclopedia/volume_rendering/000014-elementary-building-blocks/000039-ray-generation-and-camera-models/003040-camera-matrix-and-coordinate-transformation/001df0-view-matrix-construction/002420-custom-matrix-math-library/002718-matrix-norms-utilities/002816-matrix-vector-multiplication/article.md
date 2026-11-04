# Matrix-Vector Multiplication
Also called: operator* (Mat4, Vec4)
## Overview
Matrix-vector multiplication computes the linear transformation of a 4-dimensional vector by a 4x4 matrix, yielding a new 4-dimensional vector. In volume rendering, this operation is fundamental for transforming ray origins, directions, and sample points between coordinate spaces (e.g., world to view or local to world). It distinguishes itself from matrix-matrix multiplication by operating on a single vector operand, reducing computational complexity and memory bandwidth requirements.
## How it works
**Inputs and Outputs:**
- Input: A 4x4 matrix $M$ and a 4D vector $v$.
- Output: A 4D vector $r$ representing the product $M \times v$.

**Data Structures:**
The matrix $M$ is stored as a 4x4 array of elements, typically accessible via row-major or column-major indexing. The vector $v$ is a 4-element array. Let $M_{ij}$ denote the element at row $i$ and column $j$ (0-indexed), and $v_j$ denote the $j$-th component of the vector.

**Formula:**
The result vector $r$ is computed component-wise:
$$ r_i = \sum_{j=0}^{3} M_{ij} v_j $$
for $i = 0, 1, 2, 3$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mat4_vec4_mul(m: list[list[float]], v: list[float]) -> list[float]:
    """
    Computes the product of a 4x4 matrix and a 4D vector.
    
    Args:
        m: A 4x4 matrix represented as a list of 4 lists, each containing 4 floats.
           m[i][j] is the element at row i, column j.
        v: A 4D vector represented as a list of 4 floats.
        
    Returns:
        A 4D vector resulting from the multiplication.
    """
    result = [0.0] * 4
    for i in range(4):
        sum_val = 0.0
        for j in range(4):
            sum_val += m[i][j] * v[j]
        result[i] = sum_val
    return result
```

**Parameters:**
- $M$: The 4x4 transformation matrix.
- $v$: The 4D vector to be transformed.

**Edge Cases and Pitfalls:**
- **Dimension Mismatch:** The matrix must be 4x4 and the vector 4D. Using incompatible dimensions leads to undefined behavior or runtime errors.
- **Precision:** Floating-point precision issues may arise in iterative transformations. Using double precision can mitigate accumulation errors in long chains of transformations.
- **Homogeneous Coordinates:** When transforming points, the $w$ component of the result vector must be handled appropriately (e.g., perspective division) if the matrix includes projection or perspective components.

## Subtopics

### Scalar-Vector Multiplication
Multiplies a scalar by a 4D vector, scaling each component of the vector by the scalar value. It is used for uniform scaling of vectors. Parameters: a scalar value and a 4D vector. [S1]

## Operating characteristics
**Time and Memory Complexity:**
- Time: $O(1)$ with a constant number of operations (16 multiplications and 12 additions).
- Memory: $O(1)$ auxiliary space.

**Measured Performance:**
Performance depends on hardware architecture and compiler optimizations. On modern CPUs, this operation can be vectorized using SIMD instructions.

**Strengths and Limitations:**
- Strengths: Efficient, low computational overhead, essential for coordinate transformations.
- Limitations: Limited to linear transformations; cannot represent non-linear operations directly.

**When to Use:**
Use for transforming points, directions, and normals in 3D space. Avoid for non-linear transformations.

## Background
Matrix-vector multiplication is a fundamental operation in linear algebra and computer graphics. It is widely used in volume rendering for coordinate transformations. The implementation described here is based on the OpenVDB library's `nanovdb::math` namespace, which provides efficient matrix and vector operations. [S1]

## Sources
[S1] OpenVDB: nanovdb::math Namespace Reference, https://openvdb.org/documentation/doxygen/namespacenanovdb_1_1math.html, fetched 2026-10-24
