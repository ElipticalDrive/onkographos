# Matrix Norms & Utilities

Also called: MatUtils

## Overview
Matrix Norms & Utilities provides the fundamental arithmetic operations for 3x3 and 4x4 matrices used in volume rendering camera models and geometric processing. It implements norms (Frobenius, off-diagonal), transposition, and multiplication variants required for affine transformations. It differs from sibling methods like [Scaling Matrix Construction](4126-scaling-matrix-construction.md) by focusing on measurement and algebraic combination rather than specific transformation generation.

## How it works
**Inputs and Outputs:**
- Inputs: Matrices (`Mat3`, `SMat3`, `Mat4`) and vectors (`Vec3`, `Vec4`).
- Outputs: Scalar norms, transformed vectors, or new matrices.

**Data Structures:**
- `Mat3`: 9 floats in row-major order: `m00, m01, m02, m10, m11, m12, m20, m21, m22` [S1, unofficial].
- `SMat3`: 6 floats representing symmetric upper triangle: `m00, m01, m02, m11, m12, m22` [S1, unofficial].

**Formulas:**
- Frobenius norm of 3x3 matrix $A$: $||A||_F = \sqrt{\sum_{i,j} a_{ij}^2}$ [S1, unofficial].
- Frobenius norm of symmetric 3x3 matrix $A$: $||A||_F = \sqrt{a_{00}^2 + 2a_{01}^2 + 2a_{02}^2 + a_{11}^2 + 2a_{12}^2 + a_{22}^2}$ [S1, unofficial].
- Off-diagonal norm of 3x3 matrix $A$: $\text{off}(A) = \sqrt{a_{01}^2 + a_{02}^2 + a_{10}^2 + a_{12}^2 + a_{20}^2 + a_{21}^2}$ [S1, unofficial].
- Off-diagonal norm of symmetric 3x3 matrix $A$: $\text{off}(A) = \sqrt{2(a_{01}^2 + a_{02}^2 + a_{12}^2)}$ [S1, unofficial].
- Matrix multiplication $C = AB$ (3x3): $c_{ij} = \sum_{k=0}^2 a_{ik} b_{kj}$ [S1, unofficial].
- Compute $A^T A$ for 3x3 matrix $A$, storing result in symmetric matrix `Out`:
  - `Out.m00 = a.m00^2 + a.m10^2 + a.m20^2`
  - `Out.m01 = a.m00*a.m01 + a.m10*a.m11 + a.m20*a.m21`
  - `Out.m02 = a.m00*a.m02 + a.m10*a.m12 + a.m20*a.m22`
  - `Out.m11 = a.m01^2 + a.m11^2 + a.m21^2`
  - `Out.m12 = a.m01*a.m02 + a.m11*a.m12 + a.m21*a.m22`
  - `Out.m22 = a.m02^2 + a.m12^2 + a.m22^2` [S1, unofficial].
- Vector magnitude: $\|u\| = \sqrt{\sum_{i=0}^{3} u_i^2}$ [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mat3_frobenius_norm(m):
    """Computes Frobenius norm of a 3x3 matrix."""
    sum_sq = 0.0
    for i in range(3):
        for j in range(3):
            sum_sq += m[i][j] ** 2
    return math.sqrt(sum_sq)

def smat3_frobenius_norm(m):
    """Computes Frobenius norm of a symmetric 3x3 matrix (SMat3)."""
    # m layout: m00, m01, m02, m11, m12, m22
    return math.sqrt(
        m[0]**2 + 2*m[1]**2 + 2*m[2]**2 + 
        m[3]**2 + 2*m[4]**2 + m[5]**2
    )

def mat3_off_diagonal_norm(m):
    """Computes off-diagonal norm of a 3x3 matrix."""
    sum_sq = (m[0][1]**2 + m[0][2]**2 + 
              m[1][0]**2 + m[1][2]**2 + 
              m[2][0]**2 + m[2][1]**2)
    return math.sqrt(sum_sq)

def smat3_off_diagonal_norm(m):
    """Computes off-diagonal norm of a symmetric 3x3 matrix (SMat3)."""
    # m layout: m00, m01, m02, m11, m12, m22
    # Off-diagonals are m01, m02, m12
    return math.sqrt(2 * (m[1]**2 + m[2]**2 + m[4]**2))

def mat3_multiply(a, b):
    """Multiplies two 3x3 matrices A and B."""
    c = [[0.0]*3 for _ in range(3)]
    for i in range(3):
        for j in range(3):
            s = 0.0
            for k in range(3):
                s += a[i][k] * b[k][j]
            c[i][j] = s
    return c

def mat3_at_a(m):
    """Computes A^T * A for 3x3 matrix M, returns SMat3."""
    out = [0.0] * 6
    # m layout: m00, m01, m02, m10, m11, m12, m20, m21, m22
    # Indices: 0, 1, 2, 3, 4, 5, 6, 7, 8
    out[0] = m[0]**2 + m[3]**2 + m[6]**2
    out[1] = m[0]*m[1] + m[3]*m[4] + m[6]*m[7]
    out[2] = m[0]*m[2] + m[3]*m[5] + m[6]*m[8]
    out[3] = m[1]**2 + m[4]**2 + m[7]**2
    out[4] = m[1]*m[2] + m[4]*m[5] + m[7]*m[8]
    out[5] = m[2]**2 + m[5]**2 + m[8]**2
    return out

def vec4_magnitude(v):
    """Computes Euclidean norm of a 4D vector."""
    return math.sqrt(v[0]**2 + v[1]**2 + v[2]**2 + v[3]**2)

def mat4_transpose(m):
    """Returns the transpose of a 4x4 matrix."""
    t = [[0.0]*4 for _ in range(4)]
    for i in range(4):
        for j in range(4):
            t[i][j] = m[j][i]
    return t

def mat3_transpose_vec3(m, v):
    """Multiplies the transpose of a 3x3 matrix by a 3D vector."""
    # m layout: m00, m01, m02, m10, m11, m12, m20, m21, m22
    # Indices: 0, 1, 2, 3, 4, 5, 6, 7, 8
    res = [0.0, 0.0, 0.0]
    res[0] = m[0]*v[0] + m[3]*v[1] + m[6]*v[2]
    res[1] = m[1]*v[0] + m[4]*v[1] + m[7]*v[2]
    res[2] = m[2]*v[0] + m[5]*v[1] + m[8]*v[2]
    return res
```

## Subtopics

### [Matrix-Vector Multiplication](4208-matrix-vector-multiplication.md)
Computes the product of a 4x4 transformation matrix and a 4D vector, yielding a transformed 4D vector. This operation is the fundamental arithmetic step for applying affine transformations (rotation, translation, scaling, shear) to points and directions in volume rendering camera models. It differs from scalar-vector multiplication by altering direction and position via linear combination of basis vectors, rather than uniform scaling.

### [Symmetric Matrix Data Structure](6259-symmetric-matrix-data-structure.md)
The Symmetric Matrix Data Structure (SMat3) is a compact storage format for 3x3 symmetric matrices, reducing memory usage by storing only the six unique elements of the upper triangular portion. It solves the problem of redundant storage in general 3x3 matrix representations where $m_{ij} = m_{ji}$, optimizing for cache efficiency and memory bandwidth in volume rendering and geometric processing pipelines. It differs from general matrix structures by enforcing symmetry constraints at the data layout level rather than at the algorithmic level.

### Matrix Transpose (3x3)
Returns the transpose of a 4x4 matrix [S2, unofficial]. Initialize 4x4 matrix `mat`. Loop `i` from 0 to 3, `j` from 0 to 3. Set `mat[i][j] = this[j][i]`. Return `mat` [S2, unofficial].

### Matrix Transpose (3x3)
Multiplies the transpose of a 3x3 matrix by a 3d vector using 32-bit or 64-bit floating point arithmetic [S3]. Formula: `result = mat^T x xyz` [S3]. Parameters: `mat` is a pointer to an array of floats/doubles with the 3x3 matrix; `xyz` is the input vector [S3].

### Matrix Multiplication (2x3 by 3x3)
Multiplies a 2x3 matrix by a 3x3 matrix, result is a 2x3 matrix [S3]. Also supports multiplying a 3x3 matrix by a 2x3 matrix, result is a 2x3 matrix [S3]. Parameters: `lhs` (2x3 or 3x3 matrix), `rhs` (3x3 or 2x3 matrix) [S3].

### Matrix Multiplication (3x2 by 2x3)
Multiplies a 3x2 matrix by a 2x3 matrix, result is a 3x3 matrix [S3]. Parameters: `lhs` (3x2 matrix), `rhs` (2x3 matrix) [S3].

## Operating characteristics
Time complexity for norms is $O(n^2)$ for $n \times n$ matrices. Multiplication is $O(n^3)$. Memory usage for `SMat3` is 6 floats vs 9 for `Mat3` [S1, unofficial]. Performance depends on hardware floating-point unit efficiency. Precision issues may arise in large accumulations of squared terms for norms [G].

## Background
Originated in computer graphics libraries for efficient geometric processing. The specific implementations cited are from unofficial repositories and OpenVDB documentation [S1, S2, unofficial][S3].

## Sources
[S1] tuckbone/DualContouringCSharp, https://github.com/tuckbone/DualContouringCSharp, fetched 2026-10-26 (unofficial)
[S2] chloearluck/volume_rendering, https://github.com/chloearluck/volume_rendering, fetched 2026-10-24 (unofficial)
[S3] OpenVDB: nanovdb::math Namespace Reference, https://openvdb.org/documentation/doxygen/namespacenanovdb_1_1math.html, fetched 2026-10-24
