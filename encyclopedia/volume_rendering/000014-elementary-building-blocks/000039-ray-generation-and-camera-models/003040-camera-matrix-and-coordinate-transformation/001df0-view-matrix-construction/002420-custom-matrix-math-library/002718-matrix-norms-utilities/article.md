# Matrix Norms & Utilities

Also called: MatUtils

## Overview
Provides fundamental arithmetic operations for 3x3 and 4x4 matrices, including norm calculations, transposition, and multiplication variants. It serves as the computational core for [Custom Matrix Math Library](3216-custom-matrix-math-library.md), enabling the construction of view matrices and coordinate transformations without external dependencies. It distinguishes itself by offering explicit implementations for symmetric matrices (`SMat3`) to optimize memory and computation for specific geometric operations like quadratic error function (QEF) accumulation [S1, unofficial].

## How it works

### Data Structures

**`Mat3` (3x3 Matrix)**
A standard 3x3 matrix stored in row-major order as 9 floating-point numbers [S1, unofficial].
*   Layout: `m00, m01, m02, m10, m11, m12, m20, m21, m22`
*   Indexing: `m[i*3 + j]` corresponds to element at row `i`, column `j`.

**`SMat3` (Symmetric 3x3 Matrix)**
A compact storage format for symmetric matrices where $m_{ij} = m_{ji}$. It stores only the 6 unique elements of the upper triangular portion [S1, unofficial].
*   Layout: `m00, m01, m02, m11, m12, m22`
*   Mapping to full matrix:
    *   $M_{00} = \text{data}[0]$
    *   $M_{01} = M_{10} = \text{data}[1]$
    *   $M_{02} = M_{20} = \text{data}[2]$
    *   $M_{11} = \text{data}[3]$
    *   $M_{12} = M_{21} = \text{data}[4]$
    *   $M_{22} = \text{data}[5]$

### Norms

**Frobenius Norm**
Measures the magnitude of a matrix.

For a general `Mat3` $A$:
$$ ||A||_F = \sqrt{\sum_{i=0}^{2}\sum_{j=0}^{2} a_{ij}^2} $$
Implementation sums the squares of all 9 elements [S1, unofficial].

For a symmetric `SMat3` $A$, exploiting symmetry ($a_{ij} = a_{ji}$):
$$ ||A||_F = \sqrt{a_{00}^2 + 2a_{01}^2 + 2a_{02}^2 + a_{11}^2 + 2a_{12}^2 + a_{22}^2} $$
Implementation uses the 6 stored elements, doubling the off-diagonal terms before summing [S1, unofficial].

**Off-Diagonal Norm**
Measures the magnitude of the non-diagonal elements.

For a general `Mat3` $A$:
$$ \text{off}(A) = \sqrt{a_{01}^2 + a_{02}^2 + a_{10}^2 + a_{12}^2 + a_{20}^2 + a_{21}^2} $$
Implementation sums the squares of the 6 off-diagonal elements [S1, unofficial].

For a symmetric `SMat3` $A$:
$$ \text{off}(A) = \sqrt{2(a_{01}^2 + a_{02}^2 + a_{12}^2)} $$
Implementation doubles the squares of the 3 unique off-diagonal elements [S1, unofficial].

### Matrix Operations

**Matrix Multiplication ($C = AB$)**
For 3x3 matrices, computed explicitly [S1, unofficial]:
$$ c_{ij} = \sum_{k=0}^{2} a_{ik} b_{kj} $$

**Compute $A^T A$ for Symmetric Output**
Given a `Mat3` $A$, computes the symmetric matrix $S = A^T A$ into an `SMat3` `Out`. The elements are calculated as follows [S1, unofficial]:
*   `Out.m00` = $a_{00}^2 + a_{10}^2 + a_{20}^2$
*   `Out.m01` = $a_{00}a_{01} + a_{10}a_{11} + a_{20}a_{21}$
*   `Out.m02` = $a_{00}a_{02} + a_{10}a_{12} + a_{20}a_{22}$
*   `Out.m11` = $a_{01}^2 + a_{11}^2 + a_{21}^2$
*   `Out.m12` = $a_{01}a_{02} + a_{11}a_{12} + a_{21}a_{22}$
*   `Out.m22` = $a_{02}^2 + a_{12}^2 + a_{22}^2$

**Vector Magnitude**
Computes the Euclidean norm of a 4D vector $u$ [S2, unofficial]:
$$ \|u\| = \sqrt{\sum_{i=0}^{3} u_i^2} $$
Implementation: `Math.sqrt(dot(u, u))` [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mat3_frobenius_norm(m):
    """Computes Frobenius norm of a 3x3 matrix."""
    sum_sq = 0.0
    for i in range(3):
        for j in range(3):
            sum_sq += m[i][j] ** 2
    return math.sqrt(sum_sq)

def smat3_frobenius_norm(s):
    """Computes Frobenius norm of a symmetric 3x3 matrix stored in upper triangle."""
    # s = [m00, m01, m02, m11, m12, m22]
    sum_sq = (s[0]**2 + s[3]**2 + s[5]**2) + 2 * (s[1]**2 + s[2]**2 + s[4]**2)
    return math.sqrt(sum_sq)

def mat3_off_diagonal_norm(m):
    """Computes off-diagonal norm of a 3x3 matrix."""
    sum_sq = 0.0
    for i in range(3):
        for j in range(3):
            if i != j:
                sum_sq += m[i][j] ** 2
    return math.sqrt(sum_sq)

def smat3_off_diagonal_norm(s):
    """Computes off-diagonal norm of a symmetric 3x3 matrix."""
    # s = [m00, m01, m02, m11, m12, m22]
    # Off-diagonals are s[1], s[2], s[4]
    sum_sq = 2 * (s[1]**2 + s[2]**2 + s[4]**2)
    return math.sqrt(sum_sq)

def mat3_multiply(a, b):
    """Multiplies two 3x3 matrices C = A * B."""
    c = [[0.0]*3 for _ in range(3)]
    for i in range(3):
        for j in range(3):
            s = 0.0
            for k in range(3):
                s += a[i][k] * b[k][j]
            c[i][j] = s
    return c

def mat3_at_a_to_smat3(a):
    """Computes S = A^T * A and stores result in Symmetric Matrix structure."""
    # a is 3x3 matrix
    # s is SMat3: [m00, m01, m02, m11, m12, m22]
    s = [0.0] * 6
    
    # m00 = sum(a[i0]^2)
    s[0] = a[0][0]**2 + a[1][0]**2 + a[2][0]**2
    
    # m01 = dot(col0, col1)
    s[1] = a[0][0]*a[0][1] + a[1][0]*a[1][1] + a[2][0]*a[2][1]
    
    # m02 = dot(col0, col2)
    s[2] = a[0][0]*a[0][2] + a[1][0]*a[1][2] + a[2][0]*a[2][2]
    
    # m11 = sum(a[i1]^2)
    s[3] = a[0][1]**2 + a[1][1]**2 + a[2][1]**2
    
    # m12 = dot(col1, col2)
    s[4] = a[0][1]*a[0][2] + a[1][1]*a[1][2] + a[2][1]*a[2][2]
    
    # m22 = sum(a[i2]^2)
    s[5] = a[0][2]**2 + a[1][2]**2 + a[2][2]**2
    
    return s

def vec4_magnitude(v):
    """Computes Euclidean norm of a 4D vector."""
    return math.sqrt(v[0]**2 + v[1]**2 + v[2]**2 + v[3]**2)

def mat4_transpose(m):
    """Returns the transpose of a 4x4 matrix."""
    # m is 4x4 matrix
    t = [[0.0]*4 for _ in range(4)]
    for i in range(4):
        for j in range(4):
            t[i][j] = m[j][i]
    return t

def mat3_transpose_vector_mult(mat, xyz):
    """Multiplies the transpose of a 3x3 matrix by a 3D vector."""
    # mat is 3x3, xyz is 3D vector
    # result = mat^T * xyz
    res = [0.0, 0.0, 0.0]
    for i in range(3):
        s = 0.0
        for j in range(3):
            s += mat[j][i] * xyz[j] # Note: mat[j][i] is element (i,j) of transpose
        res[i] = s
    return res
```

## Subtopics

### [Matrix-Vector Multiplication](4208-matrix-vector-multiplication.md)
Computes the product of a 4x4 matrix and a 4D vector, yielding a transformed 4D vector. It performs a linear combination of the matrix columns weighted by the vector components [G].

### [Symmetric Matrix Data Structure](6259-symmetric-matrix-data-structure.md)
The Symmetric Matrix Data Structure (SMat3) is a compact storage format for 3x3 symmetric matrices, reducing memory usage by storing only the six unique elements of the upper triangular portion. It solves the problem of redundant storage in general 3x3 matrix representations where $m_{ij} = m_{ji}$, optimizing for cache efficiency and memory bandwidth in volume rendering and geometric processing pipelines. It differs from general matrix structures by enforcing symmetry constraints at the data layout level rather than at the algorithmic level.

### Matrix Transpose (3x3)
Returns the transpose of a 4x4 matrix. It initializes a 4x4 matrix `mat` and loops `i` from 0 to 3 and `j` from 0 to 3, setting `mat[i][j] = this[j][i]` [S2, unofficial].

### Matrix Transpose (3x3)
Multiplies the transpose of a 3x3 matrix by a 3D vector using 32-bit or 64-bit floating point arithmetic. The operation is defined as $result = mat^T \times xyz$, where `mat` is a pointer to an array of floats/doubles and `xyz` is the input vector [S3].

### Matrix Multiplication (3x2 by 2x3)
Multiplies a 2x3 matrix by a 3x3 matrix, resulting in a 2x3 matrix. It also supports multiplying a 3x3 matrix by a 2x3 matrix, resulting in a 2x3 matrix [S3].

### Matrix Multiplication (3x2 by 2x3)
Multiplies a 3x2 matrix by a 2x3 matrix, resulting in a 3x3 matrix [S3].

## Operating characteristics

**Time Complexity**
*   Norms: $O(1)$ for fixed-size matrices (3x3 or 4x4).
*   Multiplication: $O(n^3)$ for $n \times n$ matrices. For 3x3, this is a constant 27 multiplications and 18 additions.
*   Transpose: $O(n^2)$.

**Memory Complexity**
*   `Mat3`: 9 floats (36 bytes for 32-bit float).
*   `SMat3`: 6 floats (24 bytes for 32-bit float), a 33% reduction over `Mat3` [S1, unofficial].

**Strengths and Limitations**
*   **Strengths**: Explicit implementation avoids library overhead. `SMat3` optimization reduces memory bandwidth usage for symmetric operations common in QEF and normal estimation [S1, unofficial].
*   **Limitations**: No support for dynamic sizes. Precision depends on the underlying float/double type.

**When to Use**
*   Use for local coordinate transformations, normal transformations, and QEF assembly in volume rendering pipelines where external linear algebra libraries are unavailable or too heavy [S1, S2, unofficial].

## Background

The methods are derived from implementations in dual contouring and volume rendering projects. The `SMat3` structure and associated norms are specific to the DualContouringCSharp project, which uses them for efficient geometric processing [S1, unofficial]. The 4x4 transpose and vector magnitude utilities originate from WebGL utility libraries used in volume rendering tutorials [S2, unofficial]. The 3x3 transpose-vector multiplication and mixed-size matrix multiplications are part of the OpenVDB nanovdb math namespace, designed for high-performance grid operations [S3].

## Sources

[S1] tuckbone/DualContouringCSharp, https://github.com/tuckbone/DualContouringCSharp, fetched 2026-10-26 (unofficial)
[S2] chloearluck/volume_rendering, https://github.com/chloearluck/volume_rendering, fetched 2026-10-24 (unofficial)
[S3] OpenVDB: nanovdb::math Namespace Reference, https://openvdb.org/documentation/doxygen/namespacenanovdb_1_1math.html, fetched 2026-10-24
