# Symmetric Least Squares Solver


## Overview
The Symmetric Least Squares Solver solves the linear system $Ax = b$ for a symmetric matrix $A$ using an SVD-based pseudoinverse approach [S1, unofficial]. It distinguishes itself from the [General Least Squares Solver](4941-general-least-squares-solver.md) by operating directly on the symmetric system without explicitly forming the normal equations, thereby avoiding the condition number squaring that degrades numerical stability [S1, unofficial].

## How it works
**Inputs:**
- $A$: A symmetric matrix (stored in compact form for $3 \times 3$: $m_{00}, m_{01}, m_{02}, m_{11}, m_{12}, m_{22}$).
- $b$: A vector of the same dimension as $A$.
- $svd\_tol$: Tolerance for determining rank during eigen-decomposition.
- $svd\_sweeps$: Number of Jacobi sweeps for the eigenvalue solver.
- $pinv\_tol$: Threshold for inverting eigenvalues in the pseudoinverse step.

**Outputs:**
- $x$: The solution vector.
- $r$: The residual error $\|Ax - b\|^2$.

**Data Structures:**
- `SMat3`: Symmetric $3 \times 3$ matrix storing 6 floats: $m_{00}, m_{01}, m_{02}, m_{11}, m_{12}, m_{22}$.
- `Mat3`: General $3 \times 3$ matrix storing 9 floats: $m_{00} \dots m_{22}$.
- `Vector3`: 3-element vector with components $x, y, z$.

**Steps:**
1. **Symmetric SVD (Eigen-decomposition):** Decompose $A$ into eigenvectors $V$ and eigenvalues $D$ (stored in `VTAV`). The implementation uses a Jacobi eigenvalue algorithm with `svd_sweeps` iterations [G]. Eigenvalues smaller than `svd_tol` are treated as zero to determine the effective rank.
2. **Pseudoinverse Calculation:** Compute $P = V D^+ V^T$. For each diagonal eigenvalue $\lambda_i$ in $D$: if $\lambda_i > pinv\_tol$, set $\lambda_i^+ = 1/\lambda_i$; otherwise, set $\lambda_i^+ = 0$. Construct $D^+$ and compute the product $V D^+ V^T$ to form the pseudoinverse matrix $P$ [S1, unofficial].
3. **Solution:** Compute $x = P b$ using matrix-vector multiplication [S1, unofficial].
4. **Residual:** Compute $r = \|Ax - b\|^2$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
from typing import Tuple

class SMat3:
    def __init__(self, m00: float, m01: float, m02: float, m11: float, m12: float, m22: float):
        self.m00, self.m01, self.m02 = m00, m01, m02
        self.m11, self.m12, self.m22 = m11, m12, m22

class Mat3:
    def __init__(self, m00, m01, m02, m10, m11, m12, m20, m21, m22):
        self.m00, self.m01, self.m02 = m00, m01, m02
        self.m10, self.m11, self.m12 = m10, m11, m12
        self.m20, self.m21, self.m22 = m20, m21, m22

class Vector3:
    def __init__(self, x: float, y: float, z: float):
        self.x, self.y, self.z = x, y, z

def solve_symmetric_least_squares(A: SMat3, b: Vector3, svd_tol: float, svd_sweeps: int, pinv_tol: float) -> Tuple[Vector3, float]:
    # Step 1: Symmetric SVD / Eigen-decomposition
    V = Mat3(0,0,0, 0,0,0, 0,0,0)
    VTAV = SMat3(0,0,0, 0,0,0)
    get_symmetric_svd(A, VTAV, V, svd_tol, svd_sweeps)
    
    # Step 2: Pseudoinverse Calculation (P = V * D^+ * V^T)
    d0 = 1.0 / VTAV.m00 if VTAV.m00 > pinv_tol else 0.0
    d1 = 1.0 / VTAV.m11 if VTAV.m11 > pinv_tol else 0.0
    d2 = 1.0 / VTAV.m22 if VTAV.m22 > pinv_tol else 0.0
    
    p00 = V.m00*d0*V.m00 + V.m01*d1*V.m01 + V.m02*d2*V.m02
    p01 = V.m00*d0*V.m10 + V.m01*d1*V.m11 + V.m02*d2*V.m12
    p02 = V.m00*d0*V.m20 + V.m01*d1*V.m21 + V.m02*d2*V.m22
    p11 = V.m10*d0*V.m10 + V.m11*d1*V.m11 + V.m12*d2*V.m12
    p12 = V.m10*d0*V.m20 + V.m11*d1*V.m21 + V.m12*d2*V.m22
    p22 = V.m20*d0*V.m20 + V.m21*d1*V.m21 + V.m22*d2*V.m22
    
    P = Mat3(p00, p01, p02, p01, p11, p12, p02, p12, p22)
    
    # Step 3: Solution
    x = vmul(P, b)
    
    # Step 4: Residual
    residual = calc_error(A, x, b)
    return x, residual

def get_symmetric_svd(A: SMat3, VTAV: SMat3, V: Mat3, tol: float, sweeps: int) -> None:
    # Jacobi eigenvalue algorithm details not provided in sources [G]
    pass

def vmul(M: Mat3, v: Vector3) -> Vector3:
    return Vector3(
        M.m00*v.x + M.m01*v.y + M.m02*v.z,
        M.m10*v.x + M.m11*v.y + M.m12*v.z,
        M.m20*v.x + M.m21*v.y + M.m22*v.z
    )

def calc_error(A: SMat3, x: Vector3, b: Vector3) -> float:
    Ax = vmul(Mat3(A.m00, A.m01, A.m02, A.m01, A.m11, A.m12, A.m02, A.m12, A.m22), x)
    dx, dy, dz = Ax.x - b.x, Ax.y - b.y, Ax.z - b.z
    return dx*dx + dy*dy + dz*dz
```

**Parameters:**
- `pinv_tol`: Threshold for inverting eigenvalues in the pseudoinverse step [S1, unofficial].
- `svd_tol`: Tolerance for rank determination during eigen-decomposition.
- `svd_sweeps`: Number of Jacobi sweeps for the eigenvalue solver.

**Edge Cases:**
- **Singular Matrix:** If $A$ is singular, eigenvalues will be zero. The `pinv_tol` ensures these are not inverted, resulting in a minimum-norm solution [S1, unofficial].
- **Ill-conditioned Matrix:** Small eigenvalues may be amplified by inversion. The thresholding in `pinv_tol` stabilizes the solution [S1, unofficial].
- **Precision:** Floating-point errors in Jacobi iteration may cause $V$ to lose orthogonality. Re-orthogonalization or sufficient sweeps are required [G].

## Subtopics

### [General Least Squares Solver](4941-general-least-squares-solver.md)
Solves $Ax=b$ for non-symmetric $A$ by forming normal equations. Slower but more general.

### [Symmetric Pseudo-Inverse Calculation](5517-symmetric-pseudo-inverse-calculation.md)
The core primitive used by this solver to compute $A^+$ from eigen-decomposition.

## Operating characteristics
- **Time Complexity:** Dominated by the $3 \times 3$ eigen-decomposition, effectively $O(1)$ for fixed size.
- **Memory Complexity:** $O(1)$ for fixed size.
- **Strengths:** Direct, numerically stable for symmetric systems; handles singular/ill-conditioned matrices via pseudoinverse.
- **Limitations:** Fixed to small matrices; not scalable to large systems compared to iterative methods like CGLS.
- **When to use:** Small symmetric systems ($3 \times 3$); fallback for ill-conditioned problems; when a direct solution is preferred over iterative approximation.

## Background
The method is derived from the Dual Contouring implementation in [S1, unofficial]. It leverages the spectral theorem for symmetric matrices to provide a robust solution for local quadratic optimization problems in surface reconstruction.

## Sources
[S1] tuckbone/DualContouringCSharp, https://github.com/tuckbone/DualContouringCSharp, fetched 2026-10-26 (unofficial)
