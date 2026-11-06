# Grid Resolution Calculation

Also called: proportional grid, voxel count

## Overview
Grid Resolution Calculation determines the discrete dimensions $(N_x, N_y, N_z)$ of a uniform 3D voxel grid based on the continuous extent of the scene's bounding box and a target density or aspect ratio constraint [S1, unofficial]. It solves the problem of mapping a continuous world-space volume to a fixed-size integer array while preserving the geometric aspect ratio of the input [S1, unofficial]. It differs from adaptive methods by producing a single global resolution for the entire grid, and from fixed-resolution approaches by scaling dynamically with the scene's bounding box dimensions [S1, unofficial][S2, unofficial].

## How it works
**Inputs:**
- `size`: A 3D vector $(size.x, size.y, size.z)$ representing the physical extent (width, height, depth) of the scene's bounding box [S1, unofficial].
- `voxels_on_y`: An integer scalar representing the target number of voxels along the Y-axis [S1, unofficial].

**Outputs:**
- `grid_resolution`: A 3D integer vector $(N_x, N_y, N_z)$ representing the number of cells in each dimension [S1, unofficial].

**Data Structures:**
- `size`: `float3` or `glm::vec3`.
- `voxels_on_y`: `uint32_t` or `int`.
- `grid_resolution`: `uint32_t` or `int` vector.

**Steps:**
1. Calculate the number of voxels for the Y-axis directly from the input target:
   $$ N_y = \text{voxels\_on\_y} $$
2. Calculate the number of voxels for the X-axis by scaling the Y-target by the aspect ratio of the bounding box width to height, rounding up to the nearest integer:
   $$ N_x = \lceil \frac{\text{voxels\_on\_y}}{size.y} \times size.x \rceil $$
3. Calculate the number of voxels for the Z-axis by scaling the Y-target by the aspect ratio of the bounding box depth to height, rounding up to the nearest integer:
   $$ N_z = \lceil \frac{\text{voxels\_on\_y}}{size.y} \times size.z \rceil $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_grid_resolution(size: Vector3, voxels_on_y: int) -> Vector3:
    """
    Calculate grid resolution preserving bounding box aspect ratio.
    
    Args:
        size: Bounding box extent (x, y, z).
        voxels_on_y: Target voxel count for the Y-axis.
        
    Returns:
        Grid resolution (Nx, Ny, Nz).
    """
    ny = voxels_on_y
    
    # Calculate Nx and Nz based on aspect ratios relative to Y
    # Using ceiling to ensure the entire bounding box is covered
    nx = ceil((voxels_on_y / size.y) * size.x)
    nz = ceil((voxels_on_y / size.y) * size.z)
    
    return Vector3(nx, ny, nz)
```

**Parameters:**
- `voxels_on_y`: The primary control parameter. Its value depends on the desired memory budget and traversal efficiency [S1, unofficial].
- `size`: Derived from the scene's axis-aligned bounding box (AABB) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Division by Zero:** If `size.y` is zero or near-zero, the calculation fails. A guard clause should check for this and handle it as a degenerate bounding box [G].
- **Integer Overflow:** If `size.x` or `size.z` are extremely large relative to `size.y`, `nx` or `nz` may exceed integer limits. Use 64-bit integers for intermediate calculations if necessary [G].
- **Precision:** Floating-point division and multiplication should be performed in double precision if the bounding box dimensions vary widely in scale to prevent rounding errors that could lead to incorrect ceiling results [G].

## Subtopics

### [Grid Resolution Selection via Lambda](3270-grid-resolution-selection-via-lambda.md)
This method determines the number of cells in each dimension of a uniform 3D grid based on the scene's bounding box volume, the number of primitives, and a scaling factor $\lambda$. It solves the problem of selecting an appropriate grid resolution that balances build time and traversal efficiency by approximating cubically shaped cells, which minimize the expected ray tracing cost due to their optimal surface-area-to-volume ratio. It differs from adaptive methods by using a fixed global resolution derived from scene statistics, and from simple fixed-resolution approaches by scaling dynamically with primitive count.

### Adaptive Voxel Spacing
This variant adjusts voxel grid resolution dynamically using a quality parameter to balance performance and detail [S2, unofficial]. The quality parameter ranges from 0.0f to 3.0f, adjusted by user input (e.g., key presses) [S2, unofficial]. The voxel sizes are calculated as:
$$ \text{voxel\_size}_x = \frac{2.0f \times vw.aspx}{vw.xdim} \times (1.0f + \text{quality} \times 4.0f) $$
$$ \text{voxel\_size}_z = \frac{2.0f \times vw.aspz}{200.0f} \times (1.0f + \text{quality}) $$
where $vw.aspx$, $vw.xdim$, and $vw.aspz$ are scene-specific viewport and aspect parameters [S2, unofficial]. This method allows for real-time adjustment of grid density, unlike the static calculation in the main method [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ arithmetic operations. The calculation is independent of the number of primitives or scene complexity, relying only on bounding box dimensions and a scalar target [S1, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary memory. Only a few scalar variables are required [S1, unofficial].
- **Strengths:** Fast, deterministic, and preserves geometric aspect ratios [S1, unofficial]. Suitable for pre-computation or initialization phases [S1, unofficial].
- **Limitations:** Does not account for local geometric detail or primitive distribution, potentially leading to over-sampling in empty regions or under-sampling in dense areas [S1, unofficial]. Not suitable for scenes with highly non-uniform density [S1, unofficial].
- **When to use:** When a uniform grid is required and the bounding box is known [S1, unofficial].
- **When not to use:** When adaptive resolution or local detail preservation is needed [S1, unofficial].

## Background
The method originates from implementations such as `loryruta/voxelizer`, where it is used to initialize voxel grids for ray tracing or volume rendering [S1, unofficial]. It is based on the principle of maintaining aspect ratios to ensure isotropic voxel sizes relative to the scene's geometry [S1, unofficial]. The adaptive variant appears in holographic fractal rendering contexts, where dynamic quality adjustment is required for interactive performance [S2, unofficial].

## Sources
[S1] loryruta/voxelizer, https://github.com/loryruta/voxelizer, fetched 2026-10-27 (unofficial)
[S2] jtomori/holographic_fractals, https://github.com/jtomori/holographic_fractals, fetched 2026-10-26 (unofficial)
