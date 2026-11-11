# Constrained Jacobi Smoothing for SDFs

Also called: Constrained Jacobi Smoothing for Signed Distance Functions, constrained smoothing, energy minimization via Jacobi iteration

## Overview
Constrained Jacobi Smoothing is a technique to convert a binary volume into a smooth signed distance function (SDF) by minimizing an energy functional using Jacobi iteration, subject to constraints that preserve the binary topology [S1, unofficial]. It sets itself apart from sibling methods like Gaussian smoothing by guaranteeing the preservation of the exact binary topology (sign of the level set) and producing a valid signed distance function, whereas Gaussian smoothing may alter topology and does not guarantee an SDF [S1, unofficial].

## How it works
**Inputs and Outputs**
- Input: A binary volume array $V_{binary}$ of dimension $D \in \{2, 3\}$.
- Output: A signed distance function array $V_{sdf}$ of the same dimensions and resolution.

**Data Structures**
- **Stencil ($J$)**: A 3D convolution kernel. For 3D, it is a $5 \times 5 \times 5$ array of `float64`.
  - Center element $(2,2,2)$: $-4$.
  - Neighbors at Chebyshev distance 1 along axes (e.g., $(1,2,2)$): $-4$.
  - Neighbors at Chebyshev distance 1 along two axes (face diagonals, e.g., $(1,1,2)$): $1$.
  - All other elements: $0$.
  - For 2D, a corresponding 2D stencil is used (specific values not detailed in sources, but implied to follow similar Laplacian-like structure for 2D grid) [S1, unofficial].
- **Parameters**:
  - `jacobi_d`: Scaling factor. $1/12$ for 2D, $1/18$ for 3D [S1, unofficial].
  - `jacobi_weight`: Relaxation weight. Default $0.5$ [S1, unofficial].
  - `max_iters`: Maximum iterations. Default $500$ [S1, unofficial].

**Algorithm Steps**

1. **Initial SDF Construction**:
   Convert the binary input $V_{binary}$ into an initial signed distance field $V_{init}$.
   - Let $P$ be the set of voxels with value $1.0$ (positive/inside).
   - Let $N$ be the set of voxels with value $0.0$ (negative/outside).
   - Compute the Euclidean Distance Transform (EDT) for the positive region: $D_P = \text{EDT}(P)$.
   - Compute the Euclidean Distance Transform for the negative region: $D_N = \text{EDT}(N)$.
   - Initialize $V_{init}$:
     - For $v \in P$: $V_{init}[v] = D_P[v] - 0.5$.
     - For $v \in N$: $V_{init}[v] = -(D_N[v] - 0.5)$.
   - *Note*: The subtraction of $0.5$ centers the zero-level set between voxel centers [S1, unofficial].

2. **Bound Initialization**:
   For each voxel $v$, determine the constraints $L[v]$ (lower bound) and $U[v]$ (upper bound) based on the initial SDF $V_{init}[v]$.
   - If $V_{init}[v] < 0$:
     - $L[v] = -\infty$
     - $U[v] = V_{init}[v]$
   - If $V_{init}[v] \ge 0$:
     - $L[v] = V_{init}[v]$
     - $U[v] = \infty$

3. **Bound Clamping**:
   Refine bounds near the zero-level set to prevent instability.
   - If $|U[v]| < 1$, set $U[v] = 0$.
   - If $|L[v]| < 1$, set $L[v] = 0$.
   - *Note*: This ensures that voxels very close to the surface are constrained to stay within a unit distance of the origin, preserving the topological sign [S1, unofficial].

4. **Jacobi Iteration**:
   Repeat for $k = 0$ to `max_iters` - 1:
   - **Compute Residual**:
     Convolve the current volume $V_{curr}$ with the stencil $J$ using nearest-neighbor padding (or appropriate boundary handling for the grid). Let $R$ be the result.
     $$ R = V_{curr} * J $$
   - **Compute Update**:
     $$ \Delta V = -\text{jacobi\_d} \times R $$
   - **Weighted Update**:
     $$ V_{new}[v] = \text{jacobi\_weight} \times \Delta V[v] + (1 - \text{jacobi\_weight}) \times V_{curr}[v] $$
   - **Apply Constraints**:
     Clamp the updated value to the bounds:
     $$ V_{final}[v] = \max(L[v], \min(U[v], V_{new}[v])) $$
   - Set $V_{curr} = V_{final}$ for the next iteration.

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def constrained_jacobi_smoothing(binary_volume: np.ndarray, max_iters: int = 500) -> np.ndarray:
    """
    Converts a binary volume to a signed distance function using constrained Jacobi smoothing.
    
    Args:
        binary_volume: 2D or 3D numpy array with values 0.0 or 1.0.
        max_iters: Maximum number of Jacobi iterations.
        
    Returns:
        Signed distance function array.
    """
    if binary_volume.ndim not in (2, 3):
        raise ValueError("Input must be 2D or 3D array")

    # 1. Initial SDF Construction
    # Compute Euclidean Distance Transform for positive (1.0) and negative (0.0) regions
    # Note: Standard EDT libraries return distance to nearest non-zero. 
    # We need distance to nearest 1.0 for positive, and nearest 0.0 for negative.
    
    # For positive region: distance to nearest 1.0
    pos_mask = binary_volume == 1.0
    # For negative region: distance to nearest 0.0
    neg_mask = binary_volume == 0.0
    
    # Using scipy.ndimage.distance_transform_edt for EDT calculation
    from scipy.ndimage import distance_transform_edt
    
    dist_pos = distance_transform_edt(pos_mask)
    dist_neg = distance_transform_edt(neg_mask)
    
    # Initialize SDF
    sdf = np.zeros_like(binary_volume, dtype=np.float64)
    
    # Voxels that were 1.0 (positive)
    sdf[pos_mask] = dist_pos[pos_mask] - 0.5
    # Voxels that were 0.0 (negative)
    sdf[neg_mask] = -(dist_neg[neg_mask] - 0.5)
    
    # 2. Bound Initialization
    lower_bound = np.full_like(sdf, -np.inf)
    upper_bound = np.full_like(sdf, np.inf)
    
    # Where sdf < 0 (originally negative region)
    neg_region = sdf < 0
    lower_bound[neg_region] = -np.inf
    upper_bound[neg_region] = sdf[neg_region]
    
    # Where sdf >= 0 (originally positive region)
    pos_region = ~neg_region
    lower_bound[pos_region] = sdf[pos_region]
    upper_bound[pos_region] = np.inf
    
    # 3. Bound Clamping
    # If absolute value of upper_bound < 1, set to 0
    upper_close_zero = np.abs(upper_bound) < 1
    upper_bound[upper_close_zero] = 0.0
    
    # If absolute value of lower_bound < 1, set to 0
    lower_close_zero = np.abs(lower_bound) < 1
    lower_bound[lower_close_zero] = 0.0
    
    # 4. Jacobi Iteration
    # Define Stencil
    if binary_volume.ndim == 3:
        jacobi_d = 1.0 / 18.0
        # Create 5x5x5 stencil
        stencil = np.zeros((5, 5, 5), dtype=np.float64)
        # Center
        stencil[2, 2, 2] = -4.0
        # Neighbors at distance 1 along axes (indices 1 and 3 in 5x5x5)
        # Axes: (1,2,2), (3,2,2), (2,1,2), (2,3,2), (2,2,1), (2,2,3)
        stencil[1, 2, 2] = -4.0
        stencil[3, 2, 2] = -4.0
        stencil[2, 1, 2] = -4.0
        stencil[2, 3, 2] = -4.0
        stencil[2, 2, 1] = -4.0
        stencil[2, 2, 3] = -4.0
        # Neighbors at distance 1 along two axes (face diagonals)
        # These are indices where two coordinates are 1 or 3, one is 2
        for i in [1, 3]:
            for j in [1, 3]:
                stencil[i, j, 2] = 1.0
                stencil[i, 2, j] = 1.0
                stencil[2, i, j] = 1.0
    else: # 2D case
        jacobi_d = 1.0 / 12.0
        # 2D Stencil definition not explicitly detailed in sources, 
        # but typically follows similar Laplacian structure. 
        # Assuming a 3x3 stencil for 2D based on standard Jacobi for Laplacian.
        # However, source mentions JACOBI_R_2D. Without explicit values, 
        # we assume a standard 5-point or 9-point Laplacian scaled appropriately.
        # Given the 3D stencil is 5x5x5, the 2D equivalent might be 3x3 or 5x5.
        # Standard 2D Laplacian Jacobi uses 1/4 for center and -1/4 for neighbors.
        # The source specifies jacobi_d = 1/12. 
        # Let's assume a 3x3 stencil for 2D similar to the 3D logic if not specified.
        # Since exact 2D stencil values are not in sources, we use a comment.
        stencil = np.zeros((3, 3), dtype=np.float64) # Placeholder for 2D stencil
        # [G] Standard 2D 5-point Laplacian stencil: center -4, neighbors -1. 
        # But scaled by jacobi_d. The source implies a specific stencil JACOBI_R_2D.
        # Without explicit values, we cannot implement exact 2D. 
        # Assuming the method is primarily 3D focused in the source context.
        raise NotImplementedError("2D stencil values not provided in source [S1, unofficial]")

    jacobi_weight = 0.5
    
    current_sdf = sdf.copy()
    
    for _ in range(max_iters):
        # Convolve with stencil
        # Using scipy.ndimage.convolve for convolution
        from scipy.ndimage import convolve
        residual = convolve(current_sdf, stencil, mode='nearest')
        
        # Compute update
        update = -jacobi_d * residual
        
        # Weighted average
        new_sdf = jacobi_weight * update + (1 - jacobi_weight) * current_sdf
        
        # Apply constraints
        constrained_sdf = np.clip(new_sdf, lower_bound, upper_bound)
        
        current_sdf = constrained_sdf
        
    return current_sdf
```

**Edge Cases and Pitfalls**
- **Dimensionality**: The method requires 2D or 3D input arrays. 1D or 4D+ arrays raise a `ValueError` [S1, unofficial].
- **Boundary Conditions**: The convolution uses 'nearest' mode padding. This may introduce artifacts at the volume boundaries if the binary data does not extend to the edge or if the boundary conditions do not match the physical problem [S1, unofficial].
- **Precision**: The stencil and intermediate calculations use `float64` to maintain precision during the iterative process [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(I \cdot N)$, where $I$ is the number of iterations (default 500) and $N$ is the number of voxels. Each iteration involves a convolution and element-wise operations.
- **Memory Complexity**: $O(N)$ for storing the volume arrays and bounds. The stencil size is constant ($5^3$ for 3D).
- **Performance**: The method is iterative and can be slow for large volumes due to the high number of iterations (500) required for convergence. However, it is robust in preserving topology.
- **Strengths**: Guarantees topological preservation of the binary input. Produces a valid SDF.
- **Limitations**: Computationally expensive compared to single-pass methods. Requires an initial EDT, which can also be costly. The choice of `max_iters` affects the trade-off between accuracy and speed.
- **When to use**: When topological correctness is critical and the input is a binary volume.
- **When not to use**: When real-time performance is required or when the input is already a smooth SDF.

## Background
The method is implemented in the `PyMCubes` library (specifically in the `mcubes` module), which is a Python library for marching cubes [S1, unofficial]. The library is maintained by pmneila. The specific implementation details, including the stencil values and iteration logic, are derived from the source code of `PyMCubes` [S1, unofficial]. The technique is used to preprocess binary volumes for surface extraction using the marching cubes algorithm, ensuring that the resulting mesh is a good representation of the original binary data.

## Sources
[S1] pmneila/PyMCubes, https://github.com/pmneila/PyMCubes, fetched 2026-10-26 (unofficial)
