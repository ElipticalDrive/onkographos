# Laplacian Grid Computation
Also called: Divergence of Gradient, Laplacian Field Transformation, Laplacian of Gaussian (LoG) Scale Detection, Normalized Laplacian Detection, Second Derivative Field

## Overview
Laplacian Grid Computation transforms a scalar field $f(x,y,z)$ into a new scalar field $\nabla^2 f$ by applying the Laplace operator, which is defined as the divergence of the gradient [S1]. This operation highlights boundaries and structural features while suppressing homogeneous interiors, as areas with constant gradient change yield a Laplacian value of zero [S1]. It differs from sibling methods like [Mean Curvature Grid Computation](3126-mean-curvature-grid-computation.md) by not requiring normalization by gradient magnitude, and from [Divergence Grid Computation](3151-divergence-grid-computation.md) by operating on a scalar field's second derivatives rather than a vector field's first derivatives.

## How it works
**Inputs and Outputs:**
- Input: A scalar grid $f$ defined on a discrete domain with active voxels.
- Output: A scalar grid $L$ where each active voxel contains the Laplacian $\nabla^2 f$.

**Data Structures:**
- `Grid`: Contains voxel values, topology, and voxel spacing $(\Delta x, \Delta y, \Delta z)$.
- `Stencil`: A mechanism to access neighbor values at integer offsets relative to a center coordinate $c$ [G].

**Steps:**
1. Initialize a `GradStencil` with the input grid's topology [S2, unofficial].
2. Iterate over all active voxels $c$ in the grid tree [S2, unofficial].
3. Move the stencil to the current coordinate $c$ [S2, unofficial].
4. Compute the Laplacian scalar using the 6-point stencil (pure second derivatives along axes) [S2, unofficial].
5. Store the result in a dictionary or output grid keyed by coordinate [S2, unofficial].
6. Build the result grid using the computed data, a background value of zero, and the original grid's voxel size [S2, unofficial].

**Formulas:**
The Laplacian is the sum of the pure second partial derivatives [S1]:
$$ \nabla^2 f(x, y, z) = \frac{\partial^2 f}{\partial x^2} + \frac{\partial^2 f}{\partial y^2} + \frac{\partial^2 f}{\partial z^2} $$

Using central differences with unit spacing ($\Delta x = \Delta y = \Delta z = 1$) for the pure second derivative [G]:
$$ \frac{\partial^2 f}{\partial x^2} \approx f(x+1, y, z) - 2f(x, y, z) + f(x-1, y, z) $$

If physical voxel spacing is not unity, the differences must be divided by the squared spacing terms [G]:
$$ \frac{\partial^2 f}{\partial x^2} \approx \frac{f(x+\Delta x) - 2f(x) + f(x-\Delta x)}{\Delta x^2} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_laplacian_grid(grid):
    """
    Computes the Laplacian grid for a scalar volume.
    grid: Object with .values (dict or array), .spacing (tuple dx, dy, dz), 
          and .active_voxels (iterable of coordinates).
    Returns: A new grid object containing the Laplacian values.
    """
    dx, dy, dz = grid.spacing
    result_values = {}
    
    # Precompute inverse squared spacings for normalization
    inv_dx2 = 1.0 / (dx * dx)
    inv_dy2 = 1.0 / (dy * dy)
    inv_dz2 = 1.0 / (dz * dz)
    
    for c in grid.active_voxels:
        x, y, z = c
        
        # Access neighbors using a helper that handles boundary conditions
        # (e.g., clamping, zero-padding, or exclusion as per parent principles)
        f_center = grid.value_at(x, y, z)
        f_xp = grid.value_at(x + dx, y, z)
        f_xm = grid.value_at(x - dx, y, z)
        f_yp = grid.value_at(x, y + dy, z)
        f_ym = grid.value_at(x, y - dy, z)
        f_zp = grid.value_at(x, y, z + dz)
        f_zm = grid.value_at(x, y, z - dz)
        
        # Compute second derivatives
        d2f_dx2 = (f_xp - 2.0 * f_center + f_xm) * inv_dx2
        d2f_dy2 = (f_yp - 2.0 * f_center + f_ym) * inv_dy2
        d2f_dz2 = (f_zp - 2.0 * f_center + f_zm) * inv_dz2
        
        # Sum for Laplacian
        laplacian = d2f_dx2 + d2f_dy2 + d2f_dz2
        
        result_values[c] = laplacian
        
    # Build result grid
    return Grid(values=result_values, background=0.0, spacing=grid.spacing)
```

**Parameters and Edge Cases:**
- **Voxel Spacing**: Must be provided to normalize derivatives correctly [G].
- **Boundary Handling**: Neighbors outside the active domain require specific handling (clamping, zero-padding, or exclusion) as defined in the parent [Second Derivative Computation](6353-second-derivative-computation.md).
- **Homogeneity**: Areas where the gradient changes constantly yield $\nabla^2 f = 0$, effectively suppressing homogeneous regions [S1].

## Subtopics

### [GPU Laplacian Scale-Space Maxima Detection](5852-gpu-laplacian-scale-space-maxima-detection.md)
Detects local maxima of the Laplacian across a scale space to identify blob-like structures, merging diffusion and detection into a single pass to save GPU memory.

### [Scale Field Computation](6538-scale-field-computation.md)
Estimates a 3D scalar field encoding local feature size using iterative non-linear diffusion, allowing sub-voxel scale detection without subsampling artifacts.

### [Scale Space Construction](8275-scale-space-construction.md)
Generates a continuous multi-scale representation by solving the linear diffusion equation, extending the Laplacian operator into a temporal process.

## Operating characteristics
- **Time Complexity**: $O(N)$ where $N$ is the number of active voxels, as each voxel requires a constant number of neighbor accesses and arithmetic operations.
- **Memory Complexity**: $O(N)$ to store the output grid.
- **Strengths**: Effectively highlights boundaries and structural features while suppressing homogeneous interiors [S1]. Simple to implement using standard finite difference stencils.
- **Limitations**: Sensitive to noise, as second derivatives amplify high-frequency components. Linear diffusion can cause nearby objects to merge into a single blob before detection, leading to inaccurate scale localization if used in scale-space contexts [S3, unofficial].
- **When to use**: When boundary detection or structural feature extraction is required, or as a component in scale-space analysis.
- **When not to use**: When noise robustness is critical without prior smoothing, or when vector field divergence is needed instead of scalar Laplacian.

## Background
The Laplacian operator is defined as the divergence of the gradient [S1]. The divergence of a vector field $F$ was originally defined by a limit of the surface integral:
$$ \nabla \cdot F \equiv \lim_{V \to 0} \frac{\oint_S F \cdot ds}{V} $$
where $S$ is the closed infinitesimal boundary surface surrounding volume $V$ [S1]. In the context of scalar fields, this becomes the sum of second partial derivatives [S1]. The method is foundational for scale-space analysis, where the normalized Laplacian $t\nabla^2 L$ is used to detect blobs at multiple scales [S3, unofficial].

## Sources
[S1] Boundary-structure-aware transfer functions for volume classification, https://doi.org/10.1145/3139295.3139306, fetched 2026-10-31
[S2] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S3] Size-based Transfer Functions: A New Volume Exploration Technique, https://www.researchgate.net/publication/23456199_Size-based_Transfer_Functions_A_New_Volume_Exploration_Technique, fetched 2026-10-30 (unofficial)
