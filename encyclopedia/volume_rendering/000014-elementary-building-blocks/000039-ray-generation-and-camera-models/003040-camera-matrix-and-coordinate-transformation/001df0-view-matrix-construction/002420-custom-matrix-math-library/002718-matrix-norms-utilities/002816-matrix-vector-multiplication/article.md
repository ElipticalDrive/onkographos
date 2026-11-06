# Matrix-Vector Multiplication
Also called: operator* (Mat4, Vec4)
## Overview
Computes the product of a 4x4 transformation matrix and a 4D vector, yielding a transformed 4D vector [S1]. This operation is the fundamental arithmetic step for applying affine transformations (rotation, translation, scaling, shear) to points and directions in volume rendering camera models [S1]. It differs from scalar-vector multiplication by altering direction and position via linear combination of basis vectors, rather than uniform scaling [S1].
## How it works
**Inputs:**
- `m`: A 4x4 matrix of type `T` (typically `float` or `double`).
- `v`: A 4D vector of type `T`.

**Outputs:**
- `result`: A 4D vector of type `T` representing $m \times v$ [S1].

**Data Structures:**
The matrix `m` is stored as a 4x4 grid of elements $m_{ij}$ where $i$ is the row index (0-3) and $j$ is the column index (0-3). The vector `v` is stored as a 4-element array $v_j$ where $j$ is the component index (0-3).

**Formula:**
The result vector $r$ is computed as:
$$ r_i = \sum_{j=0}^{3} m_{ij} v_j $$
for each row $i \in \{0, 1, 2, 3\}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def matrix_vector_multiply(m: Matrix4x4[T], v: Vec4[T]) -> Vec4[T]:
    """
    Computes the product of a 4x4 matrix and a 4D vector.
    
    Args:
        m: 4x4 matrix with elements m[row][col]
        v: 4D vector with components v[0], v[1], v[2], v[3]
        
    Returns:
        Result vector r where r[i] = sum(m[i][j] * v[j] for j in 0..3)
    """
    r = Vec4[T]()
    
    # Row 0
    r[0] = m[0][0] * v[0] + m[0][1] * v[1] + m[0][2] * v[2] + m[0][3] * v[3]
    
    # Row 1
    r[1] = m[1][0] * v[0] + m[1][1] * v[1] + m[1][2] * v[2] + m[1][3] * v[3]
    
    # Row 2
    r[2] = m[2][0] * v[0] + m[2][1] * v[1] + m[2][2] * v[2] + m[2][3] * v[3]
    
    # Row 3
    r[3] = m[3][0] * v[0] + m[3][1] * v[1] + m[3][2] * v[2] + m[3][3] * v[3]
    
    return r
```

**Parameters:**
- `T`: The numeric type (e.g., `float`, `double`). Not given in the sources.

**Edge Cases and Pitfalls:**
- Precision: Accumulation of floating-point errors in the summation may occur for large matrices or vectors. Not given in the sources.
- Homogeneous Coordinates: If `v` represents a point, the resulting `w` component (index 3) may not be 1.0, requiring perspective division in subsequent steps. Not given in the sources.
- Degenerate Input: If the matrix is singular, the transformation may collapse dimensions. Not given in the sources.

## Subtopics

### [Scalar-Vector Multiplication](6274-scalar-vector-multiplication.md)
Scales a vector by a scalar factor. It is the fundamental arithmetic operation underlying the linear combination of basis vectors in matrix-vector multiplication. Unlike matrix-vector multiplication, which transforms direction and position via rotation and shear, this operation uniformly scales magnitude without altering direction (for positive scalars) or reversing it (for negative scalars). It serves as the atomic step for computing dot products, norms, and affine transformations in the OpenVDB math library.

## Operating characteristics
- Time Complexity: $O(1)$ for fixed 4x4 matrix and 4D vector, involving 16 multiplications and 12 additions.
- Memory Complexity: $O(1)$ auxiliary space.
- Measured Performance: Not given in the sources.
- Scaling: Linear with respect to vector dimension if generalized, but fixed for 4D.
- Strengths: Efficient, hardware-accelerable, fundamental for transformation pipelines.
- Limitations: Requires consistent data types; precision issues may arise in specific numerical regimes.
- When to Use: Applying camera transformations, object transformations in volume rendering.
- When Not to Use: When only scaling is needed (use scalar-vector multiplication).

## Background
Originated in the OpenVDB library's `nanovdb::math` namespace, specifically as an overloaded `operator*` for `Mat4` and `Vec4` types [S1]. It is part of the core linear algebra utilities supporting volume rendering and scientific visualization applications [S1].

## Sources
[S1] OpenVDB: nanovdb::math Namespace Reference, https://openvdb.org/documentation/doxygen/namespacenanovdb_1_1math.html, fetched 2026-10-24
