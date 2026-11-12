# Laplacian Grid Computation
Also called: Divergence of Gradient, Laplacian Field Transformation, Laplacian of Gaussian (LoG) Scale Detection, Normalized Laplacian Detection, Second Derivative Field

## Overview
Laplacian Grid Computation transforms a scalar field $f(x,y,z)$ into a new scalar field $\nabla^2 f$ by applying the Laplace operator, which is the divergence of the gradient [S1]. This operation highlights boundaries and suppresses homogeneous interiors, as areas with constant gradient change yield a constant divergence value while boundaries are differentiated from noise [S1]. It is distinct from sibling methods like [Mean Curvature Grid Computation](3126-mean-curvature-grid-computation.md) because it computes the trace of the Hessian (sum of pure second derivatives) rather than the mean curvature, and from [Divergence Grid Computation](3151-divergence-grid-computation.md) because it operates on a scalar field's gradient rather than an arbitrary vector field [S1].

## How it works
**Inputs and Outputs:**
*   Input: A scalar field $f$ defined on a 3D grid with coordinates $(x, y, z)$ and voxel spacing $\Delta x, \Delta y, \Delta z$.
*   Output: A scalar field $L$ where $L(x,y,z) = \nabla^2 f(x,y,z)$.

**Data Structures:**
*   `Grid`: A sparse or dense container mapping 3D integer coordinates $(c_x, c_y, c_z)$ to float values.
*   `Stencil`: A 6-point neighborhood structure accessing neighbors $(x\pm1, y, z), (x, y\pm1, z), (x, y, z\pm1)$.

**Formulas:**
The Laplacian is defined as the sum of pure second partial derivatives [S1]:
$$ \nabla^2 f(x, y, z) = \frac{\partial^2 f}{\partial x^2} + \frac{\partial^2 f}{\partial y^2} + \frac{\partial^2 f}{\partial z^2} $$

Using central differences with unit spacing in index space, the pure second derivatives are [S2, unofficial]:
$$ f_{xx} = f(x+1, y, z) - 2f(x, y, z) + f(x-1, y, z) $$
$$ f_{yy} = f(x, y+1, z) - 2f(x, y, z) + f(x, y-1, z) $$
$$ f_{zz} = f(x, y, z+1) - 2f(x, y, z) + f(x, y, z-1) $$

If the grid has non-unit voxel spacing, the derivatives must be scaled by the inverse square of the spacing [S2, unofficial]:
$$ \frac{\partial^2 f}{\partial x^2} \approx \frac{f_{xx}}{\Delta x^2} $$

The full Laplacian value at a voxel is:
$$ L(x,y,z) = \frac{f_{xx}}{\Delta x^2} + \frac{f_{yy}}{\Delta y^2} + \frac{f_{zz}}{\Delta z^2} $$

**Steps:**
1.  Initialize a `GradStencil` with the input grid tree [S2, unofficial].
2.  Iterate over all active voxels $(c_x, c_y, c_z)$ in the grid [S2, unofficial].
3.  Move the stencil to the current coordinate $c$ [S2, unofficial].
4.  Compute the Laplacian scalar using the stencil values and store it in a result dictionary keyed by coordinate [S2, unofficial].
5.  Build the result grid using the computed data, a background value of zero, and the original grid's voxel size [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_laplacian_grid(grid, voxel_spacing):
    """
    Computes the Laplacian grid from a scalar field.
    
    Args:
        grid: A dictionary mapping (x, y, z) tuples to float values.
        voxel_spacing: A tuple (dx, dy, dz) representing physical spacing.
        
    Returns:
        A dictionary mapping (x, y, z) tuples to Laplacian values.
    """
    dx, dy, dz = voxel_spacing
    dx_sq = dx * dx
    dy_sq = dy * dy
    dz_sq = dz * dz
    
    result = {}
    
    # Iterate over all active voxels
    for (x, y, z), f_val in grid.items():
        # Access neighbors, defaulting to 0.0 for inactive/missing voxels (zero-padding)
        f_xp1_y_z = grid.get((x + 1, y, z), 0.0)
        f_xm1_y_z = grid.get((x - 1, y, z), 0.0)
        f_x_yp1_z = grid.get((x, y + 1, z), 0.0)
        f_x_ym1_z = grid.get((x, y - 1, z), 0.0)
        f_x_y_zp1 = grid.get((x, y, z + 1), 0.0)
        f_x_y_zm1 = grid.get((x, y, z - 1), 0.0)
        
        # Compute second derivatives
        f_xx = f_xp1_y_z - 2.0 * f_val + f_xm1_y_z
        f_yy = f_x_yp1_z - 2.0 * f_val + f_x_ym1_z
        f_zz = f_x_y_zp1 - 2.0 * f_val + f_x_y_zm1
        
        # Scale by inverse square of spacing and sum
        laplacian = (f_xx / dx_sq) + (f_yy / dy_sq) + (f_zz / dz_sq)
        
        result[(x, y, z)] = laplacian
        
    return result
```

**Edge Cases and Pitfalls:**
*   **Boundary Handling:** Stencils require neighbors outside the active domain. Zero-padding (implicit background value) is the standard strategy [S2, unofficial]. Inactive voxels are typically set to zero in the output [S2, unofficial].
*   **Noise Sensitivity:** The Laplacian amplifies high-frequency noise. Using the first derivative field $f'(x)$ as input is less appropriate for general cases because extrema are not clear boundary indicators and it fails to distinguish objects with similar intensities [S1].
*   **Homogeneous Regions:** In areas where the gradient change is constant, $\nabla^2 f$ is constant, transforming continuously changed materials into homogeneous regions [S1].

## Subtopics

### [GPU Laplacian Scale-Space Maxima Detection](5852-gpu-laplacian-scale-space-maxima-detection.md)
Detects local maxima of the Laplacian across a scale space to identify blob-like structures. It merges scale-space diffusion and maxima detection into a single per-iteration pass, avoiding the storage of the full scale-space pyramid in GPU memory.

### [Scale Field Computation](6538-scale-field-computation.md)
Estimates a 3D scalar field $S$ encoding local feature size using iterative non-linear diffusion. It allows for sub-voxel scale detection and better localization without the granularity limitations of subsampled pyramids.

### [Scale Space Construction](8275-scale-space-construction.md)
Generates a continuous multi-scale representation by solving the linear diffusion equation. It avoids discretization artifacts in maxima detection by providing a continuous solution via Gaussian convolution.

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of active voxels, as each voxel requires a constant number of lookups and arithmetic operations (6 neighbors, 3 second derivatives, 3 divisions, 2 additions).
*   **Memory Complexity:** $O(N)$ to store the output grid. The input grid is read-only.
*   **Strengths:** Efficiently highlights boundaries and suppresses homogeneous interiors. Simple to implement with standard finite difference stencils.
*   **Limitations:** Sensitive to noise. Linear diffusion can cause nearby objects to merge into a single blob before detection, leading to inaccurate scale maxima localization [S3, unofficial].
*   **When to use:** When boundary detection or blob identification is required.
*   **When not to use:** When noise robustness is critical without pre-smoothing, or when non-linear diffusion is required to prevent merging of nearby features [S3, unofficial].

## Background
The Laplacian operator is the divergence of the gradient, originally defined by a limit of the surface integral [S1]. It is widely used in volume rendering for boundary-structure-aware transfer functions and scale-based exploration [S1][S3, unofficial]. The method builds on [Second Derivative Computation](6353-second-derivative-computation.md) by summing the pure second derivatives.

## Sources
[S1] Boundary-structure-aware transfer functions for volume classification, https://doi.org/10.1145/3139295.3139306, fetched 2026-10-31
[S2] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S3] Size-based Transfer Functions: A New Volume Exploration Technique, https://www.researchgate.net/publication/23456199_Size-based_Transfer_Functions_A_New_Volume_Exploration_Technique, fetched 2026-10-30 (unofficial)
