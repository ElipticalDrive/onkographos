# CGLS Solver (Conjugate Gradient Least Squares)
Also called: Sparse Linear Least Squares Solver
## Overview
The CGLS solver is a sparse linear least squares solver designed to avoid the ill-conditioning inherent in normal equations ($A^T A$) for large or rank-deficient problems [S1, unofficial]. It provides numerically stable results where direct methods fail, distinguishing itself from its sibling [Newton's Method](6442-newton-s-method.md) by using iterative conjugate gradients rather than explicit matrix inversion, and from [Gradient Descent Solver](3280-gradient-descent-solver.md) by solving the least-squares formulation directly rather than minimizing the residual via gradient steps [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: Matrix $A$ (sparse, implicit or explicit), vector $b$.
- Output: Vector $x$ minimizing $\|Ax - b\|^2$.

**Data Structures:**
- $A$: Represented as a sparse matrix (e.g., CSR format) or via a matrix-vector product function.
- $b, x, p, r$: Dense vectors of size $m$ or $n$ as appropriate.

**Algorithm Steps:**
1. Initialize $x_0 = 0$.
2. Compute initial residual $r_0 = b - A x_0$.
3. Set $p_0 = A^T r_0$.
4. Set $\rho_0 = r_0^T r_0$.
5. For $k = 0, 1, \dots$ until convergence:
   a. Compute $q_k = A p_k$.
   b. Compute $\alpha_k = \rho_k / (p_k^T A^T q_k)$.
   c. Update $x_{k+1} = x_k + \alpha_k p_k$.
   d. Update $r_{k+1} = r_k - \alpha_k q_k$.
   e. If $\|r_{k+1}\|$ is sufficiently small, terminate.
   f. Compute $\rho_{k+1} = r_{k+1}^T r_{k+1}$.
   g. Compute $\beta_k = \rho_{k+1} / \rho_k$.
   h. Update $p_{k+1} = A^T r_{k+1} + \beta_k p_k$.
   i. Increment $k$.

**Preconditioning:**
- Jacobi preconditioning is applied to achieve faster convergence [S1, unofficial]. This involves scaling the system by the inverse of the diagonal elements of $A^T A$ (or approximations thereof) before the iterative steps.

**Parameters:**
- Convergence tolerance: Not given in the sources.
- Maximum iterations: Not given in the sources.

**Edge Cases and Pitfalls:**
- Ill-conditioning: CGLS is specifically chosen to handle ill-conditioned systems where normal equations fail [S1, unofficial].
- Rank deficiency: Suitable for almost rank-deficient problems [S1, unofficial].
- Performance: Generally slower than computing $A^T A$ and $A^T b$ explicitly for moderately sized problems [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cgls_solve(A_matvec, A_T_matvec, b, max_iter=1000, tol=1e-6):
    """
    Solves min ||Ax - b||^2 using CGLS with Jacobi preconditioning.
    A_matvec: function that computes A * v
    A_T_matvec: function that computes A^T * v
    """
    n = len(b)
    x = [0.0] * n
    r = [b[i] - A_matvec(x)[i] for i in range(n)]
    p = A_T_matvec(r)
    
    # Jacobi Preconditioning: Approximate diagonal of A^T A
    # Note: Exact diagonal computation depends on A's structure.
    # Here we assume a simple diagonal approximation or identity if not provided.
    # For true Jacobi, we need diag(A^T A). 
    # Let's assume a helper `get_diagonal_AT_A` exists or is computed once.
    # If not available, standard CGLS proceeds without preconditioning.
    # The source mentions Jacobi preconditioning is included [S1, unofficial].
    # We will implement a placeholder for the preconditioner application.
    
    diag_AT_A = get_diagonal_AT_A(A_matvec, n) # [G] Helper to compute diagonal
    inv_diag = [1.0 / d if d > 1e-12 else 1.0 for d in diag_AT_A]
    
    # Preconditioned initial p
    p = [p[i] * inv_diag[i] for i in range(n)]
    
    rho = sum(r[i] * r[i] for i in range(n))
    
    for k in range(max_iter):
        q = A_matvec(p)
        Ap = A_T_matvec(q)
        pAp = sum(p[i] * Ap[i] for i in range(n))
        
        if pAp == 0:
            break
            
        alpha = rho / pAp
        x = [x[i] + alpha * p[i] for i in range(n)]
        r = [r[i] - alpha * q[i] for i in range(n)]
        
        r_norm_sq = sum(r[i] * r[i] for i in range(n))
        if r_norm_sq < tol:
            break
            
        rho_new = r_norm_sq
        beta = rho_new / rho
        
        # Preconditioned p update
        r_T = A_T_matvec(r)
        p_new = [r_T[i] * inv_diag[i] + beta * p[i] for i in range(n)]
        
        p = p_new
        rho = rho_new
        
    return x

def get_diagonal_AT_A(A_matvec, n):
    """
    Computes the diagonal of A^T A by multiplying A^T A with standard basis vectors.
    This is an O(n * nnz) operation.
    """
    diag = [0.0] * n
    for i in range(n):
        e_i = [1.0 if j == i else 0.0 for j in range(n)]
        # A * e_i is the i-th column of A
        col_i = A_matvec(e_i)
        # (A^T A)_{ii} = col_i^T * col_i
        diag[i] = sum(c * c for c in col_i)
    return diag
```
## Subtopics

### [Conjugate Gradient Solver for Compressive Volume Rendering](4270-conjugate-gradient-solver-for-compressive-volume-rendering.md)
This method recovers missing pixels in a rendered volume image by solving a regularized least-squares optimization problem using the Conjugate Gradient algorithm. It allows reconstruction from a subset of pixels, reducing total rendering time compared to full volume rendering. It differs from its parent [CGLS Solver (Conjugate Gradient Least Squares)](3285-cgls-solver-conjugate-gradient-least-squares.md) by explicitly defining the regularization term $H$ via B-spline convolution and the sampling matrix $S$ for pixel dropping, whereas the parent focuses on the general numerical stability of the least-squares formulation.

### [Symmetric Least Squares Solver](4723-symmetric-least-squares-solver.md)
The Symmetric Least Squares Solver solves the linear system $Ax = b$ where $A$ is a symmetric matrix, typically arising from the normal equations $A^T A$ in least-squares problems. It differs from the parent [CGLS Solver](3285-cgls-solver-conjugate-gradient-least-squares.md) by explicitly computing the pseudoinverse via Singular Value Decomposition (SVD) rather than using iterative conjugate gradients. This approach provides a direct solution suitable for small, dense symmetric systems (e.g., $3 \times 3$) where numerical stability via eigenvalue thresholding is required, contrasting with the iterative nature of [Conjugate Gradient Solver for Compressive Volume Rendering](4270-conjugate-gradient-solver-for-compressive-volume-rendering.md).

### [Adaptive Grid Optical Tomography](13834-adaptive-grid-optical-tomography.md)
A volumetric reconstruction method that uses an adaptive grid (octree) to represent the 3D density field, allowing for higher effective resolution and better memory efficiency than uniform grids by using fewer basis functions in uniform regions.

### Cholesky Solver
Solves $Hc = \eta$ by performing Cholesky decomposition of $H$, followed by forward and backward substitution [S2]. This is used when $H$ is positive-definite and symmetric, such as in solving for Kernel Polynomials in moment bounds [S2].

### Scalar Field Expansion Solver
Addresses problems when solving outer vertices of very sparse parent voxels that lack enough neighboring child voxels to create a linear system of equations [S3, unofficial]. It solves sparse cases using the scalar field expansion method described in Appendix B of the source [S3, unofficial]. A pitfall is that Singular Value Decomposition (SVD) often results in incorrect values for these sparse cases, with resulting values being both positive (outside bounds) and negative (inside the model) [S3, unofficial].

## Operating characteristics
- **Time Complexity:** Generally slower than computing $A^T A$ and $A^T b$ explicitly for moderately sized problems [S1, unofficial]. Convergence speed depends on the condition number of $A$; Jacobi preconditioning is used to accelerate convergence [S1, unofficial].
- **Memory Complexity:** Matrix-free implementation possible, storing only sparse structure of $A$ and vectors. Avoids forming $A^T A$, saving memory for large $A$.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Suitable for very large problems where forming $A^T A$ is infeasible or ill-conditioned [S1, unofficial].
- **Strengths:** Numerically stable for ill-conditioned and almost rank-deficient problems [S1, unofficial]. Matrix-free capability.
- **Limitations:** Slower than direct methods for small/medium well-conditioned problems [S1, unofficial]. Requires matrix-vector products.
- **When to use:** Large, sparse, ill-conditioned least-squares problems.
- **When not to use:** Small, dense, well-conditioned systems where direct solvers (like [Cholesky Solver](#cholesky-solver)) are faster and more stable.
## Background
The CGLS solver is based on the work by Paige and Saunders [PS82], as cited in the source [S1, unofficial]. It is used in volume rendering contexts, specifically for transfer function optimization, where it is evaluated against normal equations and gradient descent solvers [S1, unofficial].
## Sources
[S1] Transfer Function Optimization for Comparative Volume Rendering, https://doi.org/10.48550/arxiv.2306.05885, fetched 2026-10-26 (unofficial)
[S2] Moment Bounds are Differentiable: Efficiently Approximating Measures in Inverse Rendering, https://doi.org/10.1145/3730899, fetched 2026-10-27
[S3] Level-of-Detail Independent Voxel-Based Surface ..., https://studenttheses.uu.nl/items/a8b8233a-d142-43f1-8f6d-c0b0fe43a767, fetched 2026-10-27 (unofficial)
