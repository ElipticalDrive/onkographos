# Givens Rotation
Also called: plane rotation

## Overview
A Givens rotation is a 2D rotation applied to a $3 \times 3$ matrix to zero out or transform specific elements, used here for QR decomposition or SVD-like iterative solving [S1, unofficial]. It sets itself apart from general 3D rotation matrices by operating on a specific plane defined by two indices while leaving the third dimension unchanged, allowing for targeted element elimination in linear algebra algorithms [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A $3 \times 3$ matrix $M$, two indices $i$ and $j$ specifying the plane of rotation, and two scalars $c$ (cosine) and $s$ (sine) of the rotation angle [S1, unofficial].
- Output: The modified matrix $M'$ with elements in rows/columns $i$ and $j$ rotated [S1, unofficial].

**Data Structures:**
- Matrix $M$: A $3 \times 3$ array of floats, indexed $M_{r,c}$ for row $r$ and column $c$ (0-indexed) [S1, unofficial].

**Parameters:**
- $c$: Cosine of the rotation angle.
- $s$: Sine of the rotation angle [S1, unofficial].
- Indices $i, j$: Distinct integers from $\{0, 1, 2\}$ [G].

**Steps:**
The rotation is applied as a post-multiplication (column rotation), updating columns $i$ and $j$ for each row $k \in \{0, 1, 2\}$ [S1, unofficial]:
$$
\begin{aligned}
M'_{k,i} &= c M_{k,i} - s M_{k,j} \\
M'_{k,j} &= s M_{k,i} + c M_{k,j}
\end{aligned}
$$
All other elements $M'_{k,l}$ where $l \neq i, j$ remain unchanged ($M'_{k,l} = M_{k,l}$) [S1, unofficial].

The source provides explicit formulas for three specific cases [S1, unofficial]:
1. **rot01_post** (Plane 0-1): $m_{00}' = c m_{00} - s m_{01}$, $m_{01}' = s m_{00} + c m_{01}$, $m_{10}' = c m_{10} - s m_{11}$, $m_{11}' = s m_{10} + c m_{11}$, $m_{20}' = c m_{20} - s m_{21}$, $m_{21}' = s m_{20} + c m_{21}$, with $m_{02}, m_{12}, m_{22}$ unchanged [S1, unofficial].
2. **rot02_post** (Plane 0-2): $m_{00}' = c m_{00} - s m_{02}$, $m_{02}' = s m_{00} + c m_{02}$, $m_{10}' = c m_{10} - s m_{12}$, $m_{12}' = s m_{10} + c m_{12}$, $m_{20}' = c m_{20} - s m_{22}$, $m_{22}' = s m_{20} + c m_{22}$, with $m_{01}, m_{11}, m_{21}$ unchanged [S1, unofficial].
3. **rot12_post** (Plane 1-2): $m_{01}' = c m_{01} - s m_{02}$, $m_{02}' = s m_{01} + c m_{02}$, $m_{11}' = c m_{11} - s m_{12}$, $m_{12}' = s m_{11} + c m_{12}$, $m_{21}' = c m_{21} - s m_{22}$, $m_{22}' = s m_{21} + c m_{22}$, with $m_{00}, m_{10}, m_{20}$ unchanged [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_givens_rotation_post(matrix, i, j, c, s):
    """Applies a Givens rotation to columns i and j of a 3x3 matrix."""
    for row in range(3):
        val_i = matrix[row][i]
        val_j = matrix[row][j]
        matrix[row][i] = c * val_i - s * val_j
        matrix[row][j] = s * val_i + c * val_j
    return matrix

def compute_givens_parameters(a, b):
    # Not given in the sources; standard numerical linear algebra helper [G].
    r = (a**2 + b**2)**0.5
    if r == 0:
        return 1.0, 0.0
    return a / r, b / r
```

**Edge Cases and Pitfalls:**
- If $c^2 + s^2 \neq 1$, the transformation is not orthogonal, leading to scaling artifacts [G].
- Division by zero in parameter computation if both $a$ and $b$ are zero [G].
- Precision loss when $a \gg b$ or $b \gg a$; robust implementations use `hypot` or similar stable routines [G].

## Subtopics

### [2x2 SVD](4377-2x2-svd.md)
An analytical method to compute the Singular Value Decomposition (SVD) of a 2x2 matrix $M = U \Sigma V^T$. It sets itself apart from iterative siblings like [Symmetric Givens Rotation (Jacobi SVD)](6213-symmetric-givens-rotation-jacobi-svd.md) by providing a closed-form solution that avoids convergence loops, making it suitable for real-time or deterministic graphics pipelines.

### [Symmetric Givens Rotation (Jacobi SVD)](6213-symmetric-givens-rotation-jacobi-svd.md)
An iterative method to diagonalize a $3 \times 3$ symmetric matrix using a sequence of 2D Givens rotations, implemented via the `Schur2` and `SVD` classes in the DualContouringCSharp repository. It computes the eigenvalues and eigenvectors by zeroing off-diagonal elements.

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] tuckbone/DualContouringCSharp, https://github.com/tuckbone/DualContouringCSharp, fetched 2026-10-26 (unofficial)
