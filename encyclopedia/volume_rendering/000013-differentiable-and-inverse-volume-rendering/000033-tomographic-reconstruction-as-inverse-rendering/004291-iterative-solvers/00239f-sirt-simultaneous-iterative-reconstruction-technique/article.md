# SIRT (Simultaneous Iterative Reconstruction Technique)
Also called: Simultaneous Iterative Reconstruction Technique
## Overview
SIRT is an iterative algebraic reconstruction method that solves the linear system $Ax=y$ by normalizing ray residuals by total intersected length and backprojected updates by accumulated voxel sensitivity [S1, unofficial]. It differs from [ART](6944-algebraic-reconstruction-technique-art.md) by updating all voxels simultaneously using averaged corrections rather than sequential single-projection updates [S1, unofficial].
## How it works
**Inputs:** Projection data vector $y$, system matrix $A$ (ray-voxel intersection weights), maximum iterations $K$.
**Outputs:** Reconstructed volume vector $x$.
**Data Structures:**
- $x$: Vector of voxel values, initialized to zero.
- $R$: Diagonal matrix for ray normalization.
- $C$: Diagonal matrix for voxel sensitivity normalization.

**Steps:**
1. Initialize $x^{(0)} = 0$.
2. For $k = 0$ to $K-1$:
   a. Compute residual: $r = y - Ax^{(k)}$.
   b. Normalize residual: $r_{norm} = R r$.
   c. Backproject: $b = A^T r_{norm}$.
   d. Normalize update: $u = C b$.
   e. Update: $x^{(k+1)} = \max(0, x^{(k)} + u)$.

**Formulas:**
$$x^{(k+1)} = \max(0, x^{(k)} + CA^T R(y - Ax^{(k)}))$$
where $R$ and $C$ are diagonal matrices [S1, unofficial]:
$$R_{jj} = \begin{cases} \frac{1}{(A\mathbf{1})_j} & \text{if } (A\mathbf{1})_j > 0 \\ 0 & \text{otherwise} \end{cases}$$
$$C_{ii} = \begin{cases} \frac{1}{(A^\top\mathbf{1})_i} & \text{if } (A^\top\mathbf{1})_i > 0 \\ 0 & \text{otherwise} \end{cases}$$
$R$ normalizes each ray residual by its total intersected length; $C$ normalizes the backprojected update by accumulated voxel sensitivity [S1, unofficial]. The $\max(0, \cdot)$ enforces nonnegativity componentwise [S1, unofficial].

**Parameters:**
- Iterations $K$: Typically 200 for measured data [S1, unofficial].

**Edge Cases:**
- If $(A\mathbf{1})_j = 0$ or $(A^\top\mathbf{1})_i = 0$, the corresponding diagonal entry is 0 to avoid division by zero [S1, unofficial].
- Nonnegativity constraint prevents negative voxel values [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sirt(A, y, K):
    # A: sparse matrix or function returning Ax and A^T x
    # y: projection data vector
    # K: number of iterations
    
    # Initialize x
    x = zeros_like(y) # Assuming x and y have compatible dimensions for A
    
    # Precompute normalization factors
    # A1: sum of each row of A (total intersected length per ray)
    A1 = A.sum(axis=1)
    # A_T1: sum of each column of A (accumulated voxel sensitivity per voxel)
    A_T1 = A.sum(axis=0)
    
    # Create diagonal matrices R and C
    # R_jj = 1/A1_j if A1_j > 0 else 0
    R_diag = where(A1 > 0, 1.0 / A1, 0.0)
    # C_ii = 1/A_T1_i if A_T1_i > 0 else 0
    C_diag = where(A_T1 > 0, 1.0 / A_T1, 0.0)
    
    for k in range(K):
        # Forward projection: Ax
        Ax = A @ x
        
        # Residual: y - Ax
        residual = y - Ax
        
        # Normalize residual by R
        # R is diagonal, so element-wise multiplication
        residual_norm = R_diag * residual
        
        # Backprojection: A^T * residual_norm
        backprojected = A.T @ residual_norm
        
        # Normalize backprojected update by C
        update = C_diag * backprojected
        
        # Update x with nonnegativity constraint
        x = maximum(x + update, 0.0)
        
    return x
```
## Subtopics

### [SART (Simultaneous Algebraic Reconstruction Technique)](3277-sart-simultaneous-algebraic-reconstruction-technique.md)
SART is an iterative algebraic reconstruction technique based on the Kaczmarz projection method, used to solve the data fidelity term in tomographic reconstruction problems. It updates the reconstructed volume for each scan view, offering modest memory requirements and superior performance over direct methods like FDK in limited projection scenarios. It is preferred over Conjugate Gradient (CG) for sparse-view settings because CG suffers from numerical noise in matrix-free implementations and convergence issues with large sparse systems.

## Operating characteristics
- **Time Complexity:** $O(K \cdot N_{rays} \cdot N_{voxels})$ per iteration, where $K$ is iterations, $N_{rays}$ is number of rays, $N_{voxels}$ is number of voxels.
- **Memory Complexity:** $O(N_{rays} + N_{voxels})$ for storing $R$ and $C$ diagonals, plus storage for $A$ if explicit.
- **Performance:** Requires 200 iterations for measured data [S1, unofficial].
- **Strengths:** Enforces nonnegativity, normalizes for ray length and voxel sensitivity.
- **Limitations:** Convergence can be slow compared to direct methods like FDK in ideal scenarios [S1, unofficial].
- **When to use:** When nonnegativity is required and ray lengths vary significantly.
- **When not to use:** When speed is critical and direct methods are applicable.
## Background
SIRT is an iterative reconstruction method introduced in the context of tomographic reconstruction [S1, unofficial]. It is part of the family of algebraic reconstruction techniques.
## Sources
[S1] diffct: Differentiable CT Operators from Circular Orbits to Arbitrary Trajectories, https://doi.org/10.20944/preprints202605.1446.v2, fetched 2026-10-26 (unofficial)
