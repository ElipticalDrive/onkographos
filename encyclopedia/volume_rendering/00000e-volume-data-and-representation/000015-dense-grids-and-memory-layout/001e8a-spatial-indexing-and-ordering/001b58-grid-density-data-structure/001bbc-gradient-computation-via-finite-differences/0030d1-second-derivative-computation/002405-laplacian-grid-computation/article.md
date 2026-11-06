# Laplacian Grid Computation
Also called: Divergence of Gradient, Laplacian Field Transformation, Laplacian of Gaussian (LoG) Scale Detection, Normalized Laplacian Detection, Second Derivative Field

## Overview
Laplacian Grid Computation transforms a scalar field $f(x,y,z)$ into a new scalar field representing the Laplacian $\nabla^2 f$, which is the divergence of the gradient. This operation highlights boundaries and suppresses homogeneous interiors by detecting regions where the gradient changes [S1]. It differs from sibling methods like [Mean Curvature Grid Computation](3126-mean-curvature-grid-computation.md) by computing the trace of the Hessian (sum of second derivatives) rather than the mean curvature, and from [Divergence Grid Computation](3151-divergence-grid-computation.md) by operating on a scalar field's gradient rather than a general vector field.

## How it works
**Inputs and Outputs:**
- Input: A scalar volume grid $f$ with active voxels.
- Output: A scalar volume grid $L$ containing $\nabla^2 f$ at each active voxel.

**Data Structures:**
- `GridTree`: The spatial index structure (e.g., octree or sparse hash map) defining the domain of active voxels.
- `Stencil`: A temporary structure holding the values of the center voxel and its 6 immediate neighbors (up, down, left, right, front, back) in index space.

**Steps:**
1. Initialize a result grid $L$ with the same topology as the input grid $f$, initialized to a background value of zero.
2. Iterate over all active coordinates $c = (x, y, z)$ in the input grid's `GridTree`.
3. For each coordinate $c$, construct a local stencil by fetching the value $f(c)$ and its immediate neighbors along the three axes:
   - $f(x+1, y, z)$, $f(x-1, y, z)$
   - $f(x, y+1, z)$, $f(x, y-1, z)$
   - $f(x, y, z+1)$, $f(x, y, z-1)$
   - *Boundary Handling*: If a neighbor is inactive or out of bounds, apply clamping (repeat edge value), zero-padding, or obstacle velocity substitution as defined by the parent method's boundary rules.
4. Compute the Laplacian value using the 6-point central difference stencil:
   $$ \nabla^2 f(x, y, z) \approx [f(x+1, y, z) - 2f(x, y, z) + f(x-1, y, z)] + [f(x, y+1, z) - 2f(x, y, z) + f(x, y-1, z)] + [f(x, y, z+1) - 2f(x, y, z) + f(x, y, z-1)] $$
   Simplified:
   $$ \nabla^2 f(x, y, z) \approx f(x+1, y, z) + f(x-1, y, z) + f(x, y+1, z) + f(x, y-1, z) + f(x, y, z+1) + f(x, y, z-1) - 6f(x, y, z) $$
5. Store the computed value in $L$ at coordinate $c$.
6. Build the final result grid using the computed data, the original voxel size, and the zero background.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_laplacian_grid(input_grid, grid_tree):
    """
    Computes the Laplacian grid from a scalar volume.
    input_grid: dict mapping (x,y,z) -> float value
    grid_tree: object providing iteration over active coordinates
    """
    output_grid = {}
    
    # Define neighbor offsets for 6-point stencil
    offsets = [
        (1, 0, 0), (-1, 0, 0),
        (0, 1, 0), (0, -1, 0),
        (0, 0, 1), (0, 0, -1)
    ]
    
    for c in grid_tree.active_coordinates():
        x, y, z = c
        center_val = input_grid.get(c, 0.0) # [G] Default to 0 if inactive, assuming zero-padding or background
        
        laplacian_sum = -6.0 * center_val
        
        for dx, dy, dz in offsets:
            neighbor_coord = (x + dx, y + dy, z + dz)
            # [G] Boundary handling: fetch neighbor value, clamping or zero-padding as per parent method
            neighbor_val = input_grid.get(neighbor_coord, 0.0)
            laplacian_sum += neighbor_val
            
        output_grid[c] = laplacian_sum
        
    return output_grid
```

**Parameters:**
- Voxel spacing: Assumed to be unit spacing in index space. If physical spacing $\Delta x, \Delta y, \Delta z$ is used, the formula must be divided by the respective squared spacing terms. Not given in the sources for the basic index-space implementation.

**Edge Cases and Pitfalls:**
- **Homogeneous Regions**: In areas where the gradient is constant, $\nabla^2 f$ is zero, transforming these areas to homogeneous values [S1].
- **Noise Sensitivity**: The Laplacian is sensitive to noise. Using the first derivative field as input is less appropriate for general cases because extrema are not clear boundary indicators and fail to distinguish objects with similar intensities [S1].
- **Boundary Artifacts**: Inactive neighbors or grid edges must be resolved carefully. Incorrect boundary handling can introduce artificial boundaries or suppress real ones.

## Subtopics

### [GPU Laplacian Scale-Space Maxima Detection](5852-gpu-laplacian-scale-space-maxima-detection.md)
Detects local maxima of the Laplacian across a scale space to identify blob-like structures. It merges scale-space diffusion and maxima detection into a single pass, avoiding full scale-space pyramid storage. Choose when memory is constrained and blob detection is required.

### [Scale Field Computation](6538-scale-field-computation.md)
Estimates a 3D scalar field encoding local feature size using iterative non-linear diffusion. Choose when sub-voxel scale detection and continuous scale representation are needed for size-based transfer functions.

## Operating characteristics
- **Time Complexity**: $O(N)$, where $N$ is the number of active voxels, as each voxel requires a constant number of neighbor lookups and arithmetic operations.
- **Memory Complexity**: $O(N)$ for the output grid, plus $O(1)$ auxiliary space per voxel during iteration.
- **Performance**: The 6-point stencil is computationally inexpensive and highly parallelizable. Performance is limited by memory bandwidth when accessing neighbor values in sparse structures.
- **Strengths**: Simple implementation, effective for boundary detection, suppresses homogeneous regions.
- **Limitations**: Sensitive to noise; linear diffusion can cause nearby objects to merge into a single blob before detection, leading to inaccurate scale maxima localization [S3, unofficial]. Non-linear conductivity filters can mitigate this but add complexity.
- **When to use**: When boundary structure is the primary feature of interest, or as a component in scale-space analysis.
- **When not to use**: When noise is significant and not pre-filtered, or when preserving homogeneous internal structure is critical.

## Background
The Laplacian operator, defined as the divergence of the gradient, has origins in vector calculus and partial differential equations. In volume rendering, it is used to extract boundary structures and classify materials based on their second-order derivative properties [S1]. The concept of using the Laplacian for blob detection and scale-space analysis was formalized in the context of computer vision and scientific visualization, particularly in the work on Size-based Transfer Functions [S3, unofficial]. The method is a fundamental tool in volume data analysis for identifying features based on their local curvature properties.

## Sources
[S1] Boundary-structure-aware transfer functions for volume classification, https://doi.org/10.1145/3139295.3139306, fetched 2026-10-31
[S2] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S3] Size-based Transfer Functions: A New Volume Exploration Technique, https://www.researchgate.net/publication/23456199_Size-based_Transfer_Functions_A_New_Volume_Exploration_Technique, fetched 2026-10-30 (unofficial)
