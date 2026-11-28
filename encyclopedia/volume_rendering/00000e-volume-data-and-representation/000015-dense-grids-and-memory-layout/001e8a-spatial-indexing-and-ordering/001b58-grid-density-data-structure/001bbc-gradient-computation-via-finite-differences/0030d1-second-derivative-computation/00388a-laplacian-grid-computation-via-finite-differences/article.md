# Laplacian Grid Computation via Finite Differences

Also called: First Derivative and Laplacian Operators, Laplacian Field Transformation, Laplian Grid Computation via Finite Differences, Volume Laplacian, applyDx, applyDy

## Overview
This method computes the discrete Laplacian of a grid field by applying forward finite difference operators followed by their transposes [S1, unofficial]. Unlike standard stencils that pad boundaries with zeros, this approach constructs the operator as $D^T D$, which implicitly enforces zero-gradient boundary conditions [G]. It supports optional weighting via a matrix or scalar parameter [S1, unofficial].

## How it works
**Inputs:**
- `f`: A 2D numpy array of shape $(H, W)$ representing the scalar field [S1, unofficial].
- `gamma`: An optional weighting matrix or scalar [S1, unofficial].

**Outputs:**
- A flattened 1D array of shape $(H \times W,)$ containing the Laplacian values [S1, unofficial].

**Data Structures:**
- Intermediate derivatives are stored in arrays of shape $(H, W)$, with boundary rows or columns padded with zeros to maintain dimensions [S1, unofficial].

**Steps:**
1. **Forward Difference along Rows (Axis 0):** Compute $f_{i+1,j} - f_{i,j}$ using `np.diff(u, axis=0)`, yielding shape $(H-1, W)$, then append a zero row at the bottom to restore $(H, W)$ [S1, unofficial].
2. **Forward Difference along Columns (Axis 1):** Compute $f_{i,j+1} - f_{i,j}$ using `np.diff(u, axis=1)`, yielding shape $(H, W-1)$, then append a zero column at the right to restore $(H, W)$ [S1, unofficial].
3. **Transpose of Forward Difference along Rows:** Compute the adjoint $D_x^T v$ by negating the forward difference and adjusting boundaries: the top row is $-v_{0,j}$, middle rows are $-(v_{i,j} - v_{i-1,j})$ for $i=1 \dots H-2$, and the bottom row is $v_{H-2,j}$ [S1, unofficial].
4. **Transpose of Forward Difference along Columns:** Compute $D_y^T v$ similarly: the left column is $-v_{i,0}$, middle columns are $-(v_{i,j} - v_{i,j-1})$, and the right column is $v_{i,W-2}$ [S1, unofficial].
5. **Compute Laplacian:** Sum the adjoint-forward compositions: $L = D_x^T(D_x f) + D_y^T(D_y f)$ [S1, unofficial]. For the weighted variant, apply $\sqrt{\gamma}$ before and after each operator: $L = \sqrt{\gamma} D_x^T (\sqrt{\gamma} D_x f) + \sqrt{\gamma} D_y^T (\sqrt{\gamma} D_y f)$ [S1, unofficial]. If `gamma` is not an ndarray, it defaults to an identity matrix of size $H \times H$ [S1, unofficial].
6. **Flatten:** Return $L$ flattened to a 1D array [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def applyDx(u):
    Dx = np.diff(u, axis=0)
    return np.vstack([Dx, np.zeros((1, u.shape[1]))])

def applyDy(u):
    Dy = np.diff(u, axis=1)
    return np.hstack([Dy, np.zeros((u.shape[0], 1))])

def applyDxTrans(u):
    return np.vstack([-u[0:1, :], -np.diff(u[:-1, :], axis=0), u[-2:-1, :]])

def applyDyTrans(u):
    return np.hstack([-u[:, 0:1], -np.diff(u[:, :-1], axis=1), u[:, -2:-1]])

def laplacian(f, gamma=0):
    if not isinstance(gamma, np.ndarray):
        gamma = np.eye(f.shape[0])
    dxx = np.sqrt(gamma) @ applyDxTrans(np.sqrt(gamma) @ applyDx(f))
    dyy = np.sqrt(gamma) @ applyDyTrans(np.sqrt(gamma) @ applyDy(f))
    return (dxx + dyy).flatten()
```

**Edge Cases and Pitfalls:**
- **Gamma Matrix Multiplication:** The `@` operator performs matrix multiplication, so `gamma` must be square and compatible with the row dimension of the grid [S1, unofficial].
- **Noise Amplification:** Laplacian filtering amplifies high-frequency noise, so smoothing (e.g., Gaussian filtering) may be necessary beforehand [S2].
- **Grid Dimensions:** The slicing logic assumes grid dimensions $H, W \ge 2$ to avoid empty array concatenation errors [G].

## Subtopics

### Time Smoothness Regularization via Second Derivative
A variant used in spatiotemporal grid optimization that enforces smoothness over the time dimension by penalizing the second derivative [S3, unofficial]. It identifies time grids (indices 2, 4, 5 for non-3D grids) and computes the smoothness term as $\frac{1}{N} \sum (t_{i+2} - 2t_{i+1} + t_i)^2$ by applying the first difference operator twice along the time axis [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(H \times W)$ for the unweighted grid operations, as it requires a constant number of passes over the grid [S1, unofficial]. The weighted variant adds matrix multiplication cost, scaling with the square of the row dimension if `gamma` is dense [S1, unofficial].
- **Memory Complexity:** $O(H \times W)$ for storing intermediate derivative grids and the output [S1, unofficial].
- **Strengths:** Implicitly handles boundaries via the adjoint structure, avoiding explicit padding artifacts, and supports optional spatial weighting [S1, unofficial].
- **Limitations:** Amplifies noise without pre-smoothing, and the matrix multiplication syntax requires careful shape alignment for non-scalar `gamma` [S1, unofficial][S2].
- **When to use:** When computing discrete Laplacians on 2D grids with optional weighting in optimization pipelines [S1, unofficial].

## Background
This method originates from discrete differential operator implementations in inverse problems and image processing libraries [S1, unofficial]. The $D^T D$ construction is a standard technique in discrete differential geometry to ensure self-adjointness of the Laplacian operator [G].

## Sources
[S1] imanfs/inverse-problems-CT-scans, https://github.com/imanfs/inverse-problems-CT-scans, fetched 2026-10-27 (unofficial)
[S2] Command: volume, https://www.rbvi.ucsf.edu/chimerax/docs/user/commands/volume.html, fetched 2026-10-27
[S3] ElenaViewSynthesis/4dgs, https://github.com/ElenaViewSynthesis/4dgs, fetched 2026-10-28 (unofficial)
[S4] [2603.11045] Neural Field Thermal Tomography: A Differentiable Physics ..., https://arxiv.org/abs/2603.11045, fetched 2026-10-29 (unofficial)
