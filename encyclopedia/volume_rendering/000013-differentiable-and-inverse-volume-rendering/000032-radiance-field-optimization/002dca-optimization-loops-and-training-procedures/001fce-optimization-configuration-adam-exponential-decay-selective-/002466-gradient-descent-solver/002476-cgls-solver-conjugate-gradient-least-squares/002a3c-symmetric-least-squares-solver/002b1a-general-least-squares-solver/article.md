# General Least Squares Solver
Also called: Normal Equations Solver (via Symmetric Solver), Non-Symmetric Least Squares via Normal Equations
## Overview
The General Least Squares Solver addresses the overdetermined linear system $Ax = b$ for a non-symmetric matrix $A$ by transforming it into a symmetric positive semi-definite system via the normal equations $A^T A x = A^T b$ [S1, unofficial]. It is distinguished from its parent, the Symmetric Least Squares Solver, by its preprocessing step of explicitly forming $A^T A$ and $A^T b$, which squares the condition number of the original system, thereby gaining computational speed at the cost of numerical stability for ill-conditioned inputs [S1, unofficial].
## How it works
**Inputs:**
- $A$: A non-symmetric matrix of size $M \times N$ (typically $3 \times 3$ in the cited implementation context).
- $b$: A vector of size $M$.
- $svd\_tol$: Tolerance for Singular Value Decomposition convergence.
- $svd\_sweeps$: Number of Jacobi sweeps for SVD.
- $pinv\_tol$: Tolerance for pseudoinverse calculation (singular value cutoff).

**Outputs:**
- $x$: The solution vector of size $N$.
- Residual error (scalar).

**Data Structures:**
- `Mat3`: A $3 \times 3$ matrix structure.
- `SMat3`: A symmetric $3 \times 3$ matrix structure (stores only upper/lower triangle).
- `Vector3`: A 3-element vector.

**Steps:**
1. **Transpose**: Compute $A^T$ from $A$. Let this be $A^T$.
2. **Form Normal Matrix**: Compute $A^T A$. Since $A^T A$ is symmetric, store it in a symmetric matrix structure $ATA$.
3. **Form Right-Hand Side**: Compute $A^T b$. Let this be $b'$.
4. **Solve Symmetric System**: Call the Symmetric Least Squares Solver (SVD-based pseudoinverse) on $ATA$ and $b'$ to find $x$.
   - The Symmetric Solver computes the SVD of $ATA$, calculates the pseudoinverse, and multiplies by $b'$.
5. **Return**: Return $x$ and the residual error $\|Ax - b\|$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def solve_general_least_squares(A, b, svd_tol, svd_sweeps, pinv_tol):
    # Step 1: Transpose A
    AT = transpose(A)
    
    # Step 2: Compute ATA = A^T * A
    # Since ATA is symmetric, we can use a symmetric matrix structure
    ATA = matmul_symmetric(AT, A)
    
    # Step 3: Compute ATb = A^T * b
    ATb = vmul(AT, b)
    
    # Step 4: Solve symmetric system ATA * x = ATb using SVD pseudoinverse
    # This calls the Symmetric Least Squares Solver
    x = solve_symmetric(ATA, ATb, svd_tol, svd_sweeps, pinv_tol)
    
    # Step 5: Calculate residual error
    residual = calc_error(A, x, b)
    
    return x, residual

def solve_symmetric(ATA, ATb, svd_tol, svd_sweeps, pinv_tol):
    # Compute SVD of symmetric matrix ATA
    # ATA = V * D * V^T (since symmetric, U=V)
    V, D = get_symmetric_svd(ATA, svd_tol, svd_sweeps)
    
    # Compute pseudoinverse of ATA
    # ATA^+ = V * D^+ * V^T
    ATA_pinv = pseudoinverse_symmetric(ATA, V, D, pinv_tol)
    
    # Solve for x
    x = vmul(ATA_pinv, ATb)
    
    return x

def pseudoinverse_symmetric(ATA, V, D, pinv_tol):
    # D is diagonal matrix of singular values
    # Compute D^+ where 1/sigma if sigma > pinv_tol else 0
    D_inv = zeros_like(D)
    for i in range(len(D)):
        if D[i] > pinv_tol:
            D_inv[i] = 1.0 / D[i]
        else:
            D_inv[i] = 0.0
            
    # ATA^+ = V * D^+ * V^T
    ATA_pinv = matmul(V, matmul_diag(D_inv, transpose(V)))
    return ATA_pinv
```

**Parameters:**
- $svd\_tol$: Default not given in sources. [G] Typical value $1e-6$.
- $svd\_sweeps$: Default not given in sources. [G] Typical value 10-20 for $3 \times 3$ matrices.
- $pinv\_tol$: Default not given in sources. [G] Typical value $1e-6$.

**Edge Cases:**
- Ill-conditioned $A$: Squaring the condition number ($\kappa(A^T A) = \kappa(A)^2$) can lead to significant numerical instability and loss of precision.
- Singular $A^T A$: The pseudoinverse handles this by zeroing out components corresponding to singular values below $pinv\_tol$.

## Subtopics

### [Normal Equations Solver](4954-normal-equations-solver.md)
The Normal Equations Solver addresses the overdetermined linear system $Ax = b$ arising in radiance-field optimization by forming and solving the symmetric positive semi-definite system $A^T A x = A^T b$. It is distinguished from sibling methods like the General Least Squares Solver by explicitly squaring the condition number to gain computational speed, making it the fastest alternative for well-conditioned systems but prone to numerical instability when the matrix is ill-conditioned.

## Operating characteristics
- **Time Complexity**: Dominated by matrix multiplication $O(M^2 N)$ for forming $A^T A$ and $A^T b$, and $O(N^3)$ for SVD. For $3 \times 3$ matrices, this is constant time but faster than iterative methods like CGLS for small systems.
- **Memory Complexity**: $O(MN)$ to store $A$, $A^T$, $ATA$, and $ATb$.
- **Strengths**: Simple to implement; leverages highly optimized symmetric solvers; fast for small, well-conditioned systems.
- **Limitations**: Numerically unstable for ill-conditioned matrices due to squared condition number; not suitable for large sparse systems where forming $A^T A$ is prohibitive.
- **When to use**: Small dense systems where speed is critical and $A$ is well-conditioned.
- **When not to use**: Ill-conditioned systems or large sparse systems.

## Background
The method is derived from the standard normal equations approach to least squares, implemented in the DualContouringCSharp repository [S1, unofficial]. It builds on the Symmetric Least Squares Solver which uses SVD for robust pseudoinverse calculation.

## Sources
[S1] tuckbone/DualContouringCSharp, https://github.com/tuckbone/DualContouringCSharp, fetched 2026-10-26 (unofficial)
