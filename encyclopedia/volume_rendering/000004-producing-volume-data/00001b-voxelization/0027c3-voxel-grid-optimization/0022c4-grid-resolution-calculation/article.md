# Grid Resolution Calculation

Also called: proportional grid, voxel count

## Overview
Grid Resolution Calculation determines the dimensions $(N_x, N_y, N_z)$ of a uniform 3D voxel grid based on scene geometry and target density constraints [S1, unofficial]. It solves the problem of mapping a continuous bounding box to a discrete integer grid while preserving aspect ratios, ensuring that the resulting cells are roughly cubic to optimize subsequent traversal or rendering costs [S1, unofficial]. It differs from adaptive methods by maintaining a fixed uniform spacing across the entire domain, whereas variants like Adaptive Voxel Spacing adjust resolution dynamically based on quality parameters [S2, unofficial].

## How it works
**Inputs:**
*   `size`: A 3D vector $(size.x, size.y, size.z)$ representing the physical dimensions of the scene bounding box [S1, unofficial].
*   `voxels_on_y`: An integer specifying the desired number of voxels along the Y-axis [S1, unofficial].

**Outputs:**
*   `resolution`: A 3D integer vector $(N_x, N_y, N_z)$ representing the grid dimensions [S1, unofficial].

**Data Structures:**
*   `size`: `vec3` (float) [S1, unofficial].
*   `resolution`: `uvec3` (unsigned integer) [S1, unofficial].

**Steps:**
1.  Set the Y-axis resolution to the target value:
    $$N_y = \text{voxels\_on\_y}$$
2.  Calculate the X-axis resolution by scaling the Y-resolution by the aspect ratio of the bounding box in X and Y, then rounding up:
    $$N_x = \lceil N_y \times \frac{size.x}{size.y} \rceil$$
3.  Calculate the Z-axis resolution by scaling the Y-resolution by the aspect ratio of the bounding box in Z and Y, then rounding up:
    $$N_z = \lceil N_y \times \frac{size.z}{size.y} \rceil$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_grid_resolution(size: Vec3, voxels_on_y: int) -> UVec3:
    """
    Calculate grid resolution based on bounding box size and target Y voxels.
    
    Args:
        size: Bounding box dimensions (x, y, z).
        voxels_on_y: Target number of voxels along the Y-axis.
        
    Returns:
        Grid resolution (Nx, Ny, Nz).
    """
    ny = voxels_on_y
    
    # Calculate Nx: ceil(Ny * (size.x / size.y))
    # Handle division by zero if size.y is 0
    if size.y == 0:
        nx = 0
    else:
        nx = int(math.ceil(ny * (size.x / size.y)))
        
    # Calculate Nz: ceil(Ny * (size.z / size.y))
    if size.y == 0:
        nz = 0
    else:
        nz = int(math.ceil(ny * (size.z / size.y)))
        
    return UVec3(nx, ny, nz)
```

**Parameters:**
*   `voxels_on_y`: Typical values depend on the desired detail level and memory budget. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Zero Dimension:** If `size.y` is zero, division by zero occurs. The implementation must guard against this, typically resulting in zero or infinite resolution for X and Z, which should be handled as degenerate input [G].
*   **Precision:** Using floating-point division before ceiling ensures aspect ratio preservation. Rounding down would result in smaller cells than intended, potentially exceeding memory budgets [G].

## Subtopics

### Adaptive Voxel Spacing
This method adjusts voxel grid resolution dynamically using a quality parameter to balance performance and detail [S2, unofficial]. It differs from proportional calculation by allowing real-time adjustment of voxel size based on user input or scene complexity, rather than a fixed aspect-ratio-preserving calculation [S2, unofficial]. Choose this when interactive quality adjustment is required.
*   **Definition:** Adjusts voxel grid resolution dynamically using a quality parameter [S2, unofficial].
*   **Steps:**
    1.  Accept a quality parameter ranging from 0.0f to 3.0f, adjustable by key presses (+/-) [S2, unofficial].
    2.  Calculate X and Y voxel sizes:
        $$\text{voxel\_size}_{x,y} = \frac{2.0f \times vw.aspx}{vw.xdim} \times (1.0f + \text{quality} \times 4.0f)$$
    3.  Calculate Z voxel size:
        $$\text{voxel\_size}_z = \frac{2.0f \times vw.aspz}{200.0f} \times (1.0f + \text{quality})$$
    *   Where `vw.aspx`, `vw.xdim`, and `vw.aspz` are scene-specific viewport parameters [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ for the proportional calculation, as it involves a constant number of arithmetic operations [S1, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary space, storing only the resulting resolution vector [S1, unofficial].
*   **Strengths:** Simple to implement; guarantees cubic-ish cells which are optimal for ray tracing traversal costs due to surface-area-to-volume ratio [S1, unofficial].
*   **Limitations:** Does not account for local scene complexity; uniform resolution may waste memory in empty regions [S1, unofficial].
*   **When to use:** When a uniform grid is required and aspect ratio preservation is critical for traversal efficiency [S1, unofficial].
*   **When not to use:** When the scene has highly varying density or when interactive quality adjustment is needed [S2, unofficial].

## Background
The proportional grid calculation method originates from voxelization implementations in real-time rendering contexts, specifically within the `voxelizer` project by loryruta [S1, unofficial]. It is used in voxel-based rendering pipelines where a uniform grid is required for efficient ray marching or SVO (Sparse Voxel Octree) traversal [S1, unofficial].

## Sources
[S1] loryruta/voxelizer, https://github.com/loryruta/voxelizer, fetched 2026-10-27 (unofficial)
[S2] jtomori/holographic_fractals, https://github.com/jtomori/holographic_fractals, fetched 2026-10-26 (unofficial)
