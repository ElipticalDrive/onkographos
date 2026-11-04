# Symmetric Least Squares Solver

Also called: Symmetric SVD Solver, Symmetric Pseudo-Inverse Solver

## Overview
The Symmetric Least Squares Solver computes the minimum-norm solution to the linear system $Ax = b$ where $A$ is a symmetric matrix, using Singular Value Decomposition (SVD) to construct the Moore-Penrose pseudoinverse. It is distinguished from its parent, the [CGLS Solver](3285-cgls-solver-conjugate-gradient-least-squares.md), by its reliance on explicit spectral decomposition rather than iterative conjugate gradients, making it suitable for small, dense symmetric systems where direct inversion via SVD is computationally feasible and numerically robust against rank deficiency.

## How it works
**Inputs and Outputs:**
- Input: A symmetric $3 \times 3$ matrix $A$ (stored as 6 floats: $A_{00}, A_{01}, A_{02}, A_{11}, A_{12}, A_{22}$), a 3D vector $b$, and parameters `svd_tol`, `svd_sweeps`, `pinv_tol`.
- Output: The solution vector $x$, and the residual error $||Ax - b||^2$.

**Data Structures:**
- `SMat3`: Symmetric $3 \times 3$ matrix. Fields: `m00`, `m01`, `m02`, `m11`, `m12`, `m22`.
- `Mat3`: General $3 \times 3$ matrix. Fields: `m00`..`m22` (9 floats).
- `Vector3`: 3D vector. Fields: `x`, `y`, `z`.

**Algorithm Steps:**
1. **SVD Decomposition:** Compute the SVD of the symmetric matrix $A$. This yields the eigenvector matrix $V$ and the diagonal matrix of eigenvalues $D$ (stored as `VTAV`). The decomposition satisfies $A = V D V^T$. The implementation uses a Jacobi eigenvalue algorithm with `svd_sweeps` iterations and convergence tolerance `svd_tol` [S1, unofficial].
2. **Pseudoinverse Construction:** Compute the pseudoinverse $A^+$ using the formula $A^+ = V D^+ V^T$, where $D^+$ is the diagonal matrix with entries $D^+_{ii} = \text{pinv}(D_{ii}, \text{pinv\_tol})$.
3. **Solution Calculation:** Compute $x = A^+ b$ via matrix-vector multiplication.
4. **Error Calculation:** Return the squared residual error $||Ax - b||^2$.

**Formulas:**
- **Pseudoinverse Thresholding:**
  $$ \text{pinv}(x, \text{tol}) = \begin{cases} 0 & \text{if } |x| < \text{tol} \text{ or } |1/x| < \text{tol} \\ 1/x & \text{otherwise} \end{cases} $$
  This prevents division by zero or near-zero eigenvalues [S1, unofficial].
- **Matrix Multiplication for Pseudoinverse:**
  Given $V$ (eigenvectors) and $D$ (eigenvalues), the pseudoinverse matrix $P$ is:
  $$ P_{ij} = \sum_{k=0}^{2} V_{ik} \cdot \text{pinv}(D_{kk}, \text{tol}) \cdot V_{jk} $$
  Specifically for $3 \times 3$:
  $$ P_{00} = V_{00} D^+_0 V_{00} + V_{01} D^+_1 V_{01} + V_{02} D^+_2 V_{02} $$
  (and similarly for other elements) [S1, unofficial].
- **Residual Error:**
  $$ \text{Error} = (Ax - b) \cdot (Ax - b) $$
  where $\cdot$ is the dot product [S1, unofficial].

**Parameters:**
- `svd_tol`: Convergence tolerance for the Jacobi SVD iteration. Not given in the sources.
- `svd_sweeps`: Number of Jacobi rotation sweeps. Not given in the sources.
- `pinv_tol`: Threshold for inverting eigenvalues. Values near machine epsilon ($10^{-7}$ for float) are typical. Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def solve_symmetric_least_squares(A_sym: SMat3, b: Vector3, svd_tol: float, svd_sweeps: int, pinv_tol: float) -> (Vector3, float):
    # Step 1: SVD Decomposition
    # A_sym is symmetric. V is orthogonal matrix of eigenvectors.
    # VTAV is diagonal matrix of eigenvalues (stored as SMat3).
    V = Mat3()
    VTAV = SMat3()
    get_symmetric_svd(A_sym, VTAV, V, svd_tol, svd_sweeps)
    
    # Step 2: Compute Pseudoinverse
    pinv_matrix = compute_pseudoinverse(VTAV, V, pinv_tol)
    
    # Step 3: Solve for x
    x = mat_vec_mul(pinv_matrix, b)
    
    # Step 4: Calculate Residual Error
    residual = calc_error(A_sym, x, b)
    
    return x, residual

def get_symmetric_svd(A: SMat3, VTAV: SMat3, V: Mat3, tol: float, sweeps: int):
    """
    Computes SVD for symmetric A using Jacobi eigenvalue algorithm.
    V contains eigenvectors as columns.
    VTAV contains eigenvalues on diagonal.
    """
    # Initialize V as identity
    V.set_identity()
    
    # Copy A to VTAV
    VTAV.m00 = A.m00
    VTAV.m01 = A.m01
    VTAV.m02 = A.m02
    VTAV.m11 = A.m11
    VTAV.m12 = A.m12
    VTAV.m22 = A.m22
    
    # Perform Jacobi rotations
    for i in range(sweeps):
        # Find largest off-diagonal element
        # Apply Givens rotation to zero it out
        # Update VTAV and V
        # Check convergence against tol
        # Implementation details omitted: standard Jacobi algorithm for symmetric matrices [S1, unofficial]
        pass

def compute_pseudoinverse(D: SMat3, V: Mat3, tol: float) -> Mat3:
    """
    Computes V * D^+ * V^T
    """
    d0 = pinv(D.m00, tol)
    d1 = pinv(D.m11, tol)
    d2 = pinv(D.m22, tol)
    
    Out = Mat3()
    Out.m00 = V.m00 * d0 * V.m00 + V.m01 * d1 * V.m01 + V.m02 * d2 * V.m02
    Out.m01 = V.m00 * d0 * V.m10 + V.m01 * d1 * V.m11 + V.m02 * d2 * V.m12
    Out.m02 = V.m00 * d0 * V.m20 + V.m01 * d1 * V.m21 + V.m02 * d2 * V.m22
    Out.m10 = V.m10 * d0 * V.m00 + V.m11 * d1 * V.m01 + V.m12 * d2 * V.m02
    Out.m11 = V.m10 * d0 * V.m10 + V.m11 * d1 * V.m11 + V.m12 * d2 * V.m12
    Out.m12 = V.m10 * d0 * V.m20 + V.m11 * d1 * V.m21 + V.m12 * d2 * V.m22
    Out.m20 = V.m20 * d0 * V.m00 + V.m21 * d1 * V.m01 + V.m22 * d2 * V.m02
    Out.m21 = V.m20 * d0 * V.m10 + V.m21 * d1 * V.m11 + V.m22 * d2 * V.m12
    Out.m22 = V.m20 * d0 * V.m20 + V.m21 * d1 * V.m21 + V.m22 * d2 * V.m22
    
    return Out

def pinv(x: float, tol: float) -> float:
    if abs(x) < tol or abs(1.0 / x) < tol:
        return 0.0
    return 1.0 / x

def calc_error(A_sym: SMat3, x: Vector3, b: Vector3) -> float:
    # Convert symmetric matrix to general for multiplication
    A = Mat3()
    A.set_symmetric(A_sym)
    
    Ax = mat_vec_mul(A, x)
    diff = b - Ax
    return dot(diff, diff)
```

## Subtopics

### Symmetric Pseudo-Inverse Calculation
Computes the pseudoinverse of a symmetric matrix $A$ given its SVD components $V$ (eigenvectors) and $D$ (eigenvalues). It inverts the eigenvalues in $D$, handling near-zero values via a threshold `pinv_tol` to ensure numerical stability. It is chosen when the system is small ($3 \times 3$) and symmetric, allowing direct construction of the pseudoinverse without iterative refinement. The formula is $A^+ = V D^+ V^T$ [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ for $3 \times 3$ matrices, dominated by the fixed number of Jacobi sweeps (`svd_sweeps`).
- **Memory Complexity:** $O(1)$, storing only $3 \times 3$ matrices and vectors.
- **Strengths:** Numerically stable for rank-deficient or ill-conditioned symmetric matrices due to SVD. Direct solution avoids iterative convergence issues.
- **Limitations:** Only applicable to symmetric matrices. Computationally expensive for large matrices compared to iterative solvers like CGLS.
- **When to use:** Small, dense, symmetric linear systems in differentiable rendering pipelines where exact pseudoinverse is required.
- **When not to use:** Large sparse systems, non-symmetric matrices, or when iterative approximation is sufficient.

## Background
The method is implemented in the `DualContouringCSharp` repository by tuckbone, used for dual contouring in 3D reconstruction. It leverages SVD-based pseudoinverse for solving symmetric least squares problems, a technique common in numerical linear algebra for handling ill-conditioned systems [S1, unofficial].

## Sources
[S1] tuckbone/DualContouringCSharp, https://github.com/tuckbone/DualContouringCSharp, fetched 2026-10-26 (unofficial)
