# Constrained Jacobi Smoothing for SDFs

Also called: Constrained Jacobi Smoothing for Signed Distance Functions, constrained smoothing, energy minimization via Jacobi iteration

## Overview
Constrained Jacobi Smoothing converts a binary volume into a smooth Signed Distance Function (SDF) by minimizing an energy functional using Jacobi iteration, subject to constraints that preserve the binary topology [S1, unofficial]. It sets itself apart from Gaussian smoothing by guaranteeing that the sign of the level set (topology) is preserved and the output is a valid SDF, whereas Gaussian smoothing may alter topology and does not guarantee an SDF [S1, unofficial].

## How it works
**Inputs:** A binary volume array $V_{binary}$ (2D or 3D) [S1, unofficial].
**Outputs:** A signed distance function array $V_{sdf}$ (same dimensions as input) [S1, unofficial].

**Data Structures:**
*   $V_{sdf}$: Float array of the same shape as input.
*   $V_{lower}$: Float array for lower bounds.
*   $V_{upper}$: Float array for upper bounds.
*   $V_{residual}$: Float array for Jacobi residual.
*   $V_{update}$: Float array for step update.
*   $Stencil$: 3D array of shape $(5,5,5)$ for 3D or $(5,5)$ for 2D, containing Jacobi coefficients [S1, unofficial].

**Parameters:**
*   `max_iters`: Maximum number of Jacobi iterations. Default: 500 [S1, unofficial].
*   `jacobi_weight`: Weight for the update step. Default: 0.5 [S1, unofficial].
*   `jacobi_d`: Scaling factor.
    *   2D: $1/12$ [S1, unofficial].
    *   3D: $1/18$ [S1, unofficial].
*   $Stencil$ values (3D):
    *   Center $(0,0,0)$: $-4$ [S1, unofficial].
    *   Neighbors at distance 1 along axes (e.g., $(1,0,0)$): $-4$ [S1, unofficial].
    *   Neighbors at distance 1 along two axes (face diagonals, e.g., $(1,1,0)$): $1$ [S1, unofficial].
    *   All others: $0$ [S1, unofficial].

**Algorithm Steps:**

1.  **Initialize SDF:** Convert $V_{binary}$ to an initial SDF estimate [S1, unofficial].
    *   Create $V_{sdf}$ initialized to 0.0.
    *   For each voxel $i$:
        *   If $V_{binary}[i] > 0$:
            *   Compute Euclidean distance transform $D[i]$ for the positive region [S1, unofficial].
            *   $V_{sdf}[i] = D[i] - 0.5$ [S1, unofficial].
        *   Else:
            *   Compute Euclidean distance transform $D_{neg}[i]$ for the negative region [S1, unofficial].
            *   $V_{sdf}[i] = -(D_{neg}[i] - 0.5)$ [S1, unofficial].
2.  **Compute Bounds:** Determine constraints for each voxel to preserve topology [S1, unofficial].
    *   For each voxel $i$:
        *   If $V_{sdf}[i] < 0$:
            *   $V_{upper}[i] = V_{sdf}[i]$
            *   $V_{lower}[i] = -\infty$ [S1, unofficial].
        *   Else:
            *   $V_{upper}[i] = \infty$
            *   $V_{lower}[i] = V_{sdf}[i]$ [S1, unofficial].
3.  **Clamp Bounds:** Adjust bounds near the zero-level set [S1, unofficial].
    *   For each voxel $i$:
        *   If $|V_{upper}[i]| < 1$:
            *   $V_{upper}[i] = 0$ [S1, unofficial].
        *   If $|V_{lower}[i]| < 1$:
            *   $V_{lower}[i] = 0$ [S1, unofficial].
4.  **Iterate:** Perform Jacobi smoothing [S1, unofficial].
    *   For $k$ from 0 to `max_iters` - 1:
        *   Compute residual $V_{residual}$ by convolving $V_{sdf}$ with the appropriate stencil (2D or 3D) using nearest-neighbor boundary handling [S1, unofficial].
        *   Compute update step $V_{update} = -jacobi\_d \times V_{residual}$ [S1, unofficial].
        *   Update $V_{sdf}$:
            *   $V_{sdf} = jacobi\_weight \times V_{update} + (1 - jacobi\_weight) \times V_{sdf}$ [S1, unofficial].
        *   Apply constraints:
            *   $V_{sdf} = \max(V_{sdf}, V_{lower})$
            *   $V_{sdf} = \min(V_{sdf}, V_{upper})$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def convolve_nd(volume, stencil, mode='nearest'):
    """Convolve volume with stencil using nearest boundary handling."""
    pad_size = stencil.shape[0] // 2
    padded = np.pad(volume, pad_size, mode='constant', constant_values=0)
    result = np.zeros_like(volume)
    for idx in np.argwhere(stencil != 0):
        offset = tuple(i - pad_size for i in idx)
        slices = tuple(slice(i, i + volume.shape[d]) for d, i in enumerate(offset))
        result += stencil[tuple(idx)] * padded[slices]
    return result

def constrained_jacobi_smoothing(binary_volume, max_iters=500, jacobi_weight=0.5):
    """
    Converts a binary volume to a smooth SDF using constrained Jacobi smoothing.
    
    Args:
        binary_volume: 2D or 3D numpy array of binary values (0 or 1).
        max_iters: Maximum number of iterations.
        jacobi_weight: Weight for the Jacobi update step.
        
    Returns:
        sdf_volume: 2D or 3D numpy array of signed distance values.
    """
    if binary_volume.ndim not in (2, 3):
        raise ValueError("Input must be 2D or 3D array.")
        
    sdf_volume = np.zeros_like(binary_volume, dtype=np.float64)
    pos_mask = binary_volume > 0
    
    # Step 1: Initialize SDF
    if np.any(pos_mask):
        # Standard Euclidean distance transform [G]
        dist_pos = euclidean_distance_transform(binary_volume, target=1)
        sdf_volume[pos_mask] = dist_pos[pos_mask] - 0.5
        
    if np.any(~pos_mask):
        # Standard Euclidean distance transform [G]
        dist_neg = euclidean_distance_transform(binary_volume, target=0)
        sdf_volume[~pos_mask] = -(dist_neg[~pos_mask] - 0.5)
        
    # Step 2: Compute Bounds
    lower_bound = np.full_like(sdf_volume, -np.inf)
    upper_bound = np.full_like(sdf_volume, np.inf)
    
    lower_bound[sdf_volume >= 0] = sdf_volume[sdf_volume >= 0]
    upper_bound[sdf_volume < 0] = sdf_volume[sdf_volume < 0]
    
    # Step 3: Clamp Bounds
    upper_bound[np.abs(upper_bound) < 1] = 0
    lower_bound[np.abs(lower_bound) < 1] = 0
    
    # Step 4: Iterate
    if binary_volume.ndim == 3:
        jacobi_d = 1.0 / 18.0
        stencil = _get_3d_jacobi_stencil()
    else:
        jacobi_d = 1.0 / 12.0
        stencil = _get_2d_jacobi_stencil()
        
    for _ in range(max_iters):
        residual = convolve_nd(sdf_volume, stencil, mode='nearest')
        update = -jacobi_d * residual
        sdf_volume = jacobi_weight * update + (1 - jacobi_weight) * sdf_volume
        sdf_volume = np.maximum(sdf_volume, lower_bound)
        sdf_volume = np.minimum(sdf_volume, upper_bound)
        
    return sdf_volume

def _get_3d_jacobi_stencil():
    """Returns the 5x5x5 3D Jacobi stencil."""
    stencil = np.zeros((5, 5, 5), dtype=np.float64)
    center = (2, 2, 2)
    stencil[center] = -4.0
    
    axes_neighbors = [
        (3, 2, 2), (1, 2, 2),
        (2, 3, 2), (2, 1, 2),
        (2, 2, 3), (2, 2, 1)
    ]
    for idx in axes_neighbors:
        stencil[idx] = -4.0
        
    face_diag_neighbors = [
        (3, 3, 2), (3, 1, 2), (1, 3, 2), (1, 1, 2),
        (3, 2, 3), (3, 2, 1), (1, 2, 3), (1, 2, 1),
        (2, 3, 3), (2, 3, 1), (2, 1, 3), (2, 1, 1)
    ]
    for idx in face_diag_neighbors:
        stencil[idx] = 1.0
        
    return stencil

def _get_2d_jacobi_stencil():
    """Returns the 5x5 2D Jacobi stencil."""
    stencil = np.zeros((5, 5), dtype=np.float64)
    center = (2, 2)
    stencil[center] = -4.0
    
    axes_neighbors = [
        (3, 2), (1, 2),
        (2, 3), (2, 1)
    ]
    for idx in axes_neighbors:
        stencil[idx] = -4.0
        
    face_diag_neighbors = [
        (3, 3), (3, 1), (1, 3), (1, 1)
    ]
    for idx in face_diag_neighbors:
        stencil[idx] = 1.0
        
    return stencil

def euclidean_distance_transform(volume, target):
    """
    Computes Euclidean distance transform for voxels equal to target [G].
    Returns distance to nearest voxel NOT equal to target.
    Standard implementation omitted; typically uses BFS or chamfer masks.
    """
    pass
```

## Operating characteristics
*   **Time Complexity**: Depends on the number of iterations (`max_iters`) and the cost of convolution. Each iteration involves a convolution with a $5 \times 5 \times 5$ (3D) or $5 \times 5$ (2D) stencil [S1, unofficial].
*   **Memory Complexity**: Requires storage for the SDF volume, lower/upper bound volumes, residual, and update arrays. Total memory is roughly $5 \times N$ where $N$ is the number of voxels [S1, unofficial].
*   **Strengths**: Preserves binary topology (sign of the level set) and produces a valid SDF [S1, unofficial].
*   **Limitations**: Requires 2D or 3D input; 1D or 4D+ arrays raise a ValueError [S1, unofficial]. Convergence speed depends on `max_iters` and `jacobi_d` [S1, unofficial].
*   **When to use**: When topology preservation is critical and a smooth SDF is required from a binary volume [S1, unofficial].
*   **When not to use**: When topology changes are acceptable or faster, less constrained smoothing is sufficient [S1, unofficial].

## Background
The method is used to generate smooth SDFs from binary volumes for use in rendering and geometric processing [S1, unofficial].

## Sources
[S1] pmneila/PyMCubes, https://github.com/pmneila/PyMCubes, fetched 2026-10-26 (unofficial)
