# Symmetric Least Squares Solver
## Overview
The Symmetric Least Squares Solver solves the linear system $Ax = b$ for a symmetric matrix $A$ using an SVD-based pseudoinverse [S1, unofficial]. It returns the solution vector $x$ and the squared residual error $\|Ax - b\|^2$ [S1, unofficial].

## How it works
**Inputs:**
- $A$: A symmetric matrix (stored as upper triangular elements).
- $b$: A vector of the same dimension as $A$.
- $svd\_tol$: Tolerance for SVD convergence or singular value classification.
- $svd\_sweeps$: Number of iterations for the SVD algorithm.
- $pinv\_tol$: Threshold for inverting eigenvalues; eigenvalues below this magnitude are treated as zero to ensure numerical stability [S1, unofficial].

**Outputs:**
- $x$: The solution vector.
- $residual$: The squared residual error $\|Ax - b\|^2$ [S1, unofficial].

**Data Structures:**
- `SMat3`: A symmetric $3 \times 3$ matrix stored with 6 floats: `m00, m01, m02, m11, m12, m22` [S1, unofficial].
- `Mat3`: A general $3 \times 3$ matrix stored with 9 floats [S1, unofficial].
- `Vector3`: A 3-element vector [G].

**Steps:**
1. **SVD Decomposition**: Call `getSymmetricSvd(A, VTAV, V, svd_tol, svd_sweeps)` to compute the eigenvectors $V$ and the diagonalized eigenvalues $VTAV$ [S1, unofficial].
2. **Pseudoinverse Calculation**: Call `pseudoinverse(out pinv, VTAV, V, pinv_tol)` to compute the pseudoinverse matrix $A^+$ by inverting the eigenvalues in $VTAV$ and thresholding those below `pinv_tol` [S1, unofficial].
3. **Solution**: Compute $x = A^+ b$ using matrix-vector multiplication [S1, unofficial].
4. **Residual**: Calculate and return $\|Ax - b\|^2$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def solve_symmetric_least_squares(A_upper_tri, b, svd_tol, svd_sweeps, pinv_tol):
    # Step 1: SVD Decomposition
    # getSymmetricSvd computes eigenvectors V and eigenvalues for symmetric A
    # Not given in the sources: specific iterative diagonalization routine.
    V, eigenvalues = get_symmetric_svd(A_upper_tri, svd_tol, svd_sweeps)
    
    # Step 2: Compute Pseudoinverse
    # Invert eigenvalues, thresholding near-zero values
    lambda_inv = [1.0 / lam if abs(lam) > pinv_tol else 0.0 for lam in eigenvalues]
    
    # A_plus = V * diag(lambda_inv) * V^T
    pinv = compute_pseudoinverse_matrix(V, lambda_inv)
    
    # Step 3: Solve for x
    x = mat_vec_mul(pinv, b)
    
    # Step 4: Compute Residual Error ||Ax - b||^2
    residual = calc_error(A_upper_tri, x, b)
    
    return x, residual

def get_symmetric_svd(A_upper_tri, tol, sweeps):
    """
    Computes eigenvectors V and eigenvalues for symmetric A.
    Not given in the sources: specific Givens/Jacobi rotation logic.
    Returns: (V: 3x3 matrix, eigenvalues: list of 3 floats)
    """
    # Not given in the sources: specific iterative diagonalization routine.
    pass

def compute_pseudoinverse_matrix(V, lambda_inv):
    """
    Computes V * diag(lambda_inv) * V^T explicitly.
    V: 3x3 matrix (list of lists)
    lambda_inv: list of 3 floats
    Returns: 3x3 matrix
    """
    pinv = [[0.0] * 3 for _ in range(3)]
    for i in range(3):
        for j in range(3):
            val = 0.0
            for k in range(3):
                val += V[i][k] * lambda_inv[k] * V[j][k]
            pinv[i][j] = val
    return pinv

def mat_vec_mul(M, v):
    """Matrix-vector multiplication M * v"""
    n = len(M)
    result = [0.0] * n
    for i in range(n):
        for j in range(n):
            result[i] += M[i][j] * v[j]
    return result

def calc_error(A_upper_tri, x, b):
    """Computes ||Ax - b||^2"""
    A = upper_tri_to_full(A_upper_tri)
    Ax = mat_vec_mul(A, x)
    diff = [Ax[i] - b[i] for i in range(3)]
    return sum(d * d for d in diff)

def upper_tri_to_full(upper_tri):
    """Converts [m00, m01, m02, m11, m12, m22] to 3x3 matrix"""
    m00, m01, m02, m11, m12, m22 = upper_tri
    return [
        [m00, m01, m02],
        [m01, m11, m12],
        [m02, m12, m22]
    ]
```

**Parameters:**
- `pinv_tol`: Threshold for inverting eigenvalues. Eigenvalues with magnitude less than this are treated as zero [S1, unofficial]. Typical value not given in the sources.
- `svd_tol`: Tolerance for SVD convergence. Typical value not given in the sources.
- `svd_sweeps`: Number of sweeps for the SVD algorithm. Typical value not given in the sources.

**Edge Cases and Pitfalls:**
- **Singular/Ill-conditioned Matrices**: The `pinv_tol` parameter handles near-zero eigenvalues, preventing division by zero and stabilizing the solution [S1, unofficial].
- **Symmetry Assumption**: The method assumes $A$ is symmetric, as the decomposition and storage format rely on this property [S1, unofficial].
- **Dimensionality**: The implementation is specific to $3 \times 3$ matrices (`SMat3`, `Vector3`) [S1, unofficial].

## Subtopics

### [General Least Squares Solver](4941-general-least-squares-solver.md)
Solves the overdetermined linear system $Ax = b$ for a non-symmetric matrix $A$ by explicitly forming the normal equations $A^T A x = A^T b$. Choose when $A$ is not symmetric.

### [Symmetric Pseudo-Inverse Calculation](5517-symmetric-pseudo-inverse-calculation.md)
Computes the pseudoinverse of a symmetric matrix from its eigenvectors and diagonal eigenvalues, handling near-zero eigenvalues via a threshold to ensure numerical stability. This is the core linear algebra primitive used by the Symmetric Least Squares Solver.

### [GPU Volume Reconstruction via Conjugate Gradient](14433-gpu-volume-reconstruction-via-conjugate-gradient.md)
Reconstructs spatially varying volume properties by solving the ill-conditioned linear system $Wa = o$ using Conjugate Gradient minimization on the GPU, avoiding explicit storage of the large sparse matrix $W$. Choose for large-scale volume rendering where explicit SVD is too expensive.

## Operating characteristics
- **Strengths**: Provides a direct solution for symmetric systems; handles singular or ill-conditioned matrices via eigenvalue thresholding [S1, unofficial].
- **Limitations**: Explicitly forms the pseudoinverse, which is computationally expensive for large matrices; limited to symmetric matrices [S1, unofficial].
- **When to Use**: Small, dense, symmetric linear systems (e.g., local optimization steps in dual contouring) where stability is critical [S1, unofficial].
- **When Not to Use**: Large, sparse systems where iterative methods are preferred [S1, unofficial].

## Background
The method is implemented in the `DualContouringCSharp` repository by tuckbone [S1, unofficial]. It is used in the context of dual contouring for implicit surface reconstruction, where small symmetric linear systems arise from optimizing vertex positions [S1, unofficial].

## Sources
[S1] tuckbone/DualContouringCSharp, https://github.com/tuckbone/DualContouringCSharp, fetched 2026-10-26 (unofficial)
