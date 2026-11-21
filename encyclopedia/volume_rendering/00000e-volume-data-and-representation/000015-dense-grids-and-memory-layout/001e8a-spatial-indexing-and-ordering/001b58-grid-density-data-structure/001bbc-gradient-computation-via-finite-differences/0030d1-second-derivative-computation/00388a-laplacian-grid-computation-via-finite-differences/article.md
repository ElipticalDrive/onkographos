# Laplacian Grid Computation via Finite Differences

Also called: First Derivative and Laplacian Operators, Laplacian Field Transformation, applyDx, applyDy

## Overview
This method computes the discrete Laplacian of a 2D grid field using forward and backward finite difference operators, specifically designed for use in inverse problems and regularization where the adjoint structure is critical. Unlike the standard 5-point stencil described in [Second Derivative Computation](6353-second-derivative-computation.md), this approach explicitly constructs the Laplacian matrix as $D^T D$ (where $D$ is the first derivative operator), which implicitly enforces Neumann-like boundary conditions (zero gradient at boundaries) rather than zero-padding. It is primarily used in CT reconstruction and image processing pipelines [S1, unofficial].

## How it works
The method operates on a 2D grid $f$ of shape $(H, W)$. It computes the Laplacian $\nabla^2 f$ by applying forward differences followed by their transposes (backward differences). It supports both unweighted and weighted (gamma-weighted) variants.

**Inputs:**
- `f`: A 2D numpy array of shape $(H, W)$ representing the scalar field.
- `gamma`: (Optional) A weighting matrix or scalar. If scalar, it is treated as a uniform weight. If not provided or not an ndarray, uniform weighting is assumed [S1, unofficial].

**Outputs:**
- A flattened 1D array of shape $(H \times W,)$ containing the Laplacian values.

**Data Structures:**
- The intermediate derivatives are stored in arrays of the same shape as `f` or slightly modified to accommodate boundary padding.

**Steps:**

1.  **Compute Forward Difference along Rows (Axis 0):**
    Apply `np.diff` along axis 0. This computes $f_{i+1,j} - f_{i,j}$. The result has shape $(H-1, W)$. Pad with a zero row at the bottom to restore shape $(H, W)$.
    $$ D_x f = \begin{bmatrix} f_{1,0}-f_{0,0} & \dots & f_{1,W-1}-f_{0,W-1} \\ \vdots & \ddots & \vdots \\ f_{H-1,W-1}-f_{H-2,W-1} \\ 0 & \dots & 0 \end{bmatrix} $$

2.  **Compute Forward Difference along Columns (Axis 1):**
    Apply `np.diff` along axis 1. This computes $f_{i,j+1} - f_{i,j}$. The result has shape $(H, W-1)$. Pad with a zero column at the right to restore shape $(H, W)$.
    $$ D_y f = \begin{bmatrix} f_{0,1}-f_{0,0} & \dots & f_{0,W-1}-f_{0,W-2} & 0 \\ \vdots & \ddots & \vdots & \vdots \\ f_{H-1,1}-f_{H-1,0} & \dots & f_{H-1,W-1}-f_{H-1,W-2} & 0 \end{bmatrix} $$

3.  **Compute Backward Difference (Transpose of Forward) along Rows:**
    The transpose operator $D_x^T$ is implemented by negating the forward difference and handling boundaries.
    - Top row: $-f_{0,j}$
    - Middle rows: $-(f_{i,j} - f_{i-1,j})$ for $i=1 \dots H-2$
    - Bottom row: $f_{H-2,j}$
    $$ D_x^T v = \begin{bmatrix} -v_{0,0} & \dots & -v_{0,W-1} \\ -(v_{1,0}-v_{0,0}) & \dots & -(v_{1,W-1}-v_{0,W-1}) \\ \vdots & \ddots & \vdots \\ -(v_{H-2,j}-v_{H-3,j}) & \dots & -(v_{H-2,W-1}-v_{H-3,W-1}) \\ v_{H-2,0} & \dots & v_{H-2,W-1} \end{bmatrix} $$
    *Note: The source implementation uses slicing `u[0:1, :]`, `u[:-1, :]`, and `u[-2:-1, :]` to achieve this efficiently.*

4.  **Compute Backward Difference (Transpose of Forward) along Columns:**
    Similarly for $D_y^T$:
    - Left column: $-f_{i,0}$
    - Middle columns: $-(f_{i,j} - f_{i,j-1})$
    - Right column: $f_{i,W-2}$

5.  **Compute Laplacian:**
    - **Unweighted:** $L = D_x^T (D_x f) + D_y^T (D_y f)$.
    - **Weighted:** If `gamma` is provided, apply weights before and after the derivative operators. Let $W = \sqrt{\gamma}$. Then $L = W D_x^T (W D_x f) + W D_y^T (W D_y f)$. If `gamma` is not an ndarray, it is replaced by an identity matrix of size $H \times W$ (or treated as scalar multiplication if broadcastable, but the source code checks `isinstance(gamma, np.ndarray)` and sets `gamma = np.eye(f.shape[0])` if false, which implies a specific handling for scalar inputs in the matrix multiplication context, likely assuming $f$ is flattened or gamma is diagonal [S1, unofficial]).

6.  **Flatten:**
    Return $L$ flattened to 1D.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def applyDx(u):
    """Applies the first derivative along the first dimension (rows).
    Computes u[i+1, j] - u[i, j] and pads with zeros at the bottom."""
    Dx = np.diff(u, axis=0)
    # Append a zero row at the bottom to maintain shape (H, W)
    zero_row = np.zeros((1, u.shape[1]))
    Dx = np.vstack([Dx, zero_row])
    return Dx

def applyDy(u):
    """Applies the first derivative along the second dimension (columns).
    Computes u[i, j+1] - u[i, j] and pads with zeros at the right."""
    Dy = np.diff(u, axis=1)
    # Append a zero column at the right to maintain shape (H, W)
    zero_col = np.zeros((u.shape[0], 1))
    Dy = np.hstack([Dy, zero_col])
    return Dy

def applyDxTrans(u):
    """Applies the transpose of the first derivative along the first dimension (rows).
    Implements D^T v = -D v with boundary adjustments."""
    # Top row: -u[0, :]
    # Middle rows: -(u[i, :] - u[i-1, :]) for i=1 to H-2
    # Bottom row: u[H-2, :]
    # Note: np.diff(u[:-1, :], axis=0) computes u[1:]-u[0:]
    DxT = np.vstack([
        -u[0:1, :],
        -np.diff(u[:-1, :], axis=0),
        u[-2:-1, :]
    ])
    return DxT

def applyDyTrans(u):
    """Applies the transpose of the first derivative along the second dimension (columns).
    Implements D^T v = -D v with boundary adjustments."""
    # Left col: -u[:, 0]
    # Middle cols: -(u[:, j] - u[:, j-1])
    # Right col: u[:, W-2]
    DyT = np.hstack([
        -u[:, 0:1],
        -np.diff(u[:, :-1], axis=1),
        u[:, -2:-1]
    ])
    return DyT

def laplacian(f, gamma=0):
    """Computes the Laplacian of grid f.
    
    Args:
        f: 2D numpy array (H, W)
        gamma: Weighting matrix or scalar. If 0 or not ndarray, uses uniform weights.
        
    Returns:
        Flattened 1D numpy array of Laplacian values.
    """
    # Handle gamma weighting
    if not isinstance(gamma, np.ndarray):
        # If gamma is not an array (e.g., scalar 0 or 1), treat as uniform
        # The source sets gamma = np.eye(f.shape[0]) if not ndarray, 
        # but then uses np.sqrt(gamma) @ ... which implies matrix mult.
        # If gamma is scalar, we assume uniform weighting (identity operation).
        # However, the source code specifically checks isinstance.
        # If gamma is 0 (default), it falls into this block.
        # To match source behavior for scalar gamma=0:
        gamma_mat = np.eye(f.shape[0]) # This might be a bug in source if f is 2D and gamma is scalar 0
        # But typically for uniform Laplacian, we just skip weighting.
        # Let's follow the source logic strictly:
        # If gamma is not ndarray, it sets gamma = np.eye(f.shape[0]).
        # Then np.sqrt(gamma) is Identity.
        # So effectively, if gamma is not ndarray, it uses Identity weighting.
        pass
    else:
        gamma_mat = gamma

    # Compute weighted derivatives
    sqrt_gamma = np.sqrt(gamma_mat)
    
    # Apply D_x with weights: sqrt(G) * D_x * sqrt(G) * f
    # Note: The source does: np.sqrt(gamma) @ applyDxTrans(np.sqrt(gamma)@applyDx(f))
    # This implies matrix multiplication. If f is 2D, this requires f to be flattened 
    # or gamma to be compatible. The source code calls laplacian(f) where f is 2D.
    # np.sqrt(gamma) @ applyDx(f) will fail if shapes don't align for matmul.
    # Assuming f is flattened in the actual usage context or gamma is diagonal.
    # For the purpose of this article, we assume the operations are element-wise 
    # if gamma is diagonal, or matrix mult if gamma is full.
    # Given the source uses `@`, it expects matrix multiplication.
    
    # Unweighted case (gamma=0 or not ndarray -> Identity)
    # If gamma is identity, sqrt(gamma) is identity.
    
    # Compute D_x f
    Dx_f = applyDx(f)
    # Apply weight: sqrt(G) @ Dx_f
    # If gamma is identity, this is just Dx_f
    weighted_Dx_f = np.sqrt(gamma_mat) @ Dx_f if gamma_mat.shape[0] == Dx_f.shape[0] else Dx_f # Guard for shape
    
    # Compute D_x^T (weighted_Dx_f)
    DxT_weighted = applyDxTrans(weighted_Dx_f)
    
    # Apply weight again: sqrt(G) @ DxT_weighted
    dxx = np.sqrt(gamma_mat) @ DxT_weighted if gamma_mat.shape[0] == DxT_weighted.shape[0] else DxT_weighted
    
    # Same for y
    Dy_f = applyDy(f)
    weighted_Dy_f = np.sqrt(gamma_mat) @ Dy_f if gamma_mat.shape[0] == Dy_f.shape[0] else Dy_f
    DyT_weighted = applyDyTrans(weighted_Dy_f)
    dyy = np.sqrt(gamma_mat) @ DyT_weighted if gamma_mat.shape[0] == DyT_weighted.shape[0] else DyT_weighted
    
    laplace_f = dxx + dyy
    
    return laplace_f.flatten()
```

**Edge Cases and Pitfalls:**
- **Shape Mismatch:** The matrix multiplication `np.sqrt(gamma) @ ...` requires `gamma` to be square and compatible with the flattened or row-major representation of the grid. If `f` is 2D, direct matrix multiplication with a 2D `gamma` (if `gamma` is meant to be a 2D weight map) will fail unless `gamma` is diagonal or `f` is flattened first. The source code's check `if not isinstance(gamma, np.ndarray)` suggests it handles scalars by defaulting to identity, but the matrix multiplication syntax `@` is risky for 2D arrays unless reshaped [S1, unofficial].
- **Boundary Conditions:** The padding with zeros in `applyDx`/`applyDy` and the specific boundary handling in `applyDxTrans`/`applyDyTrans` ensure that the operator is self-adjoint ($D^T$ is the true adjoint of $D$ under the Euclidean inner product). This avoids the need for explicit boundary value assumptions but implies zero-gradient boundaries.
- **Gamma Handling:** If `gamma` is a scalar (e.g., 1), the source code sets it to `np.eye(f.shape[0])`. This creates an identity matrix of size $H \times H$. If $f$ is $(H, W)$, `np.sqrt(gamma) @ applyDx(f)` will fail because `applyDx(f)` is $(H-1, W)$ or $(H, W)$, and matrix multiplication requires inner dimensions to match. The source code likely assumes `f` is flattened to $(N,)$ before calling `laplacian` or that `gamma` is applied element-wise. The pseudocode above includes a guard, but in practice, one must ensure `gamma` is either a scalar (handled by the `if` block) or a diagonal matrix matching the flattened size [S1, unofficial].

## Subtopics

### Time Smoothness Regularization via Second Derivative
A variant used in spatiotemporal grid optimization (e.g., 4D Gaussian Splatting) that enforces smoothness over the time dimension. It identifies time grids (indices 2, 4, 5 for non-3D grids) and computes the second derivative along the time axis. The smoothness term is $\frac{1}{N} \sum (t_{i+2} - 2t_{i+1} + t_i)^2$, computed by applying the first difference operator twice [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(H \times W)$ for the 2D Laplacian, as it involves a constant number of passes over the grid. The weighted version adds matrix multiplication cost, which is $O(N^2)$ if `gamma` is dense and $N=H \times W$, but typically `gamma` is diagonal or scalar, making it $O(N)$.
- **Memory Complexity:** $O(H \times W)$ for storing intermediate derivative grids.
- **Strengths:** Implicitly handles boundaries via the adjoint structure, avoiding explicit padding artifacts. Efficient for large grids.
- **Limitations:** The matrix multiplication syntax in the source code can be ambiguous for 2D inputs. Not suitable for 3D volumes without extension (the source is 2D-specific).
- **When to use:** When implementing regularizers in inverse problems (like CT reconstruction) where the adjoint property is crucial for gradient-based optimization [S1, unofficial].

## Background
This method is derived from implementations in inverse problems and CT scan reconstruction libraries, specifically the `inverse-problems-CT-scans` repository [S1, unofficial]. It is also referenced in the context of 4D Gaussian Splatting for time-smoothness regularization [S2, unofficial]. The approach of using $D^T D$ for Laplacian computation is a standard technique in discrete differential geometry and image processing to ensure self-adjointness.

## Sources
[S1] imanfs/inverse-problems-CT-scans, https://github.com/imanfs/inverse-problems-CT-scans, fetched 2026-10-27 (unofficial)
[S2] ElenaViewSynthesis/4dgs, https://github.com/ElenaViewSynthesis/4dgs, fetched 2026-10-28 (unofficial)
