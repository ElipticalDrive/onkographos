# Grid Resolution Calculation
Also called: proportional grid, voxel count
## Overview
Calculates the integer dimensions of a 3D voxel grid based on a target number of voxels on the Y-axis and the aspect ratio of the scene bounding box [S1, unofficial]. It serves as the foundational step for determining grid capacity before optimization or occupancy counting [S1, unofficial]. It is distinguished from its siblings, such as [Grid Resolution Selection via Lambda](3270-grid-resolution-selection-via-lambda.md), by its explicit reliance on direct geometric division of the bounding box dimensions rather than a scaling factor derived from primitive counts [S1, unofficial][S2, unofficial].
## How it works
**Inputs:**
- `voxels_on_y`: Integer target number of voxels along the Y-axis.
- `size`: `vec3` representing the width (X), height (Y), and depth (Z) of the scene's axis-aligned bounding box (AABB).

**Outputs:**
- `grid_dims`: `uvec3` containing the calculated number of voxels for X, Y, and Z dimensions.

**Data Structures:**
- `vec3`: 3-component floating-point vector.
- `uvec3`: 3-component unsigned integer vector.

**Steps:**
1. Set the Y-dimension of the grid resolution to the input target:
   $$Grid_Y = \text{voxels\_on\_y}$$
2. Calculate the X-dimension by scaling the Y-target by the aspect ratio of the bounding box's X and Y dimensions, then rounding up:
   $$Grid_X = \lceil \frac{\text{voxels\_on\_y}}{\text{size}.y} \times \text{size}.x \rceil$$
3. Calculate the Z-dimension by scaling the Y-target by the aspect ratio of the bounding box's Z and Y dimensions, then rounding up:
   $$Grid_Z = \lceil \frac{\text{voxels\_on\_y}}{\text{size}.y} \times \text{size}.z \rceil$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_grid_resolution(voxels_on_y: int, size: Vector3) -> Vector3i:
    """
    Calculate 3D grid resolution based on Y-axis voxel count and bounding box aspect ratios.
    
    Args:
        voxels_on_y: Target number of voxels along the Y-axis.
        size: Bounding box dimensions (x, y, z).
        
    Returns:
        Vector3i: Grid dimensions (x, y, z).
    """
    # Guard against division by zero
    if size.y == 0.0:
        return Vector3i(0, voxels_on_y, 0)

    # Calculate Y dimension directly
    grid_y = voxels_on_y

    # Calculate X dimension: scale by aspect ratio x/y, round up
    # ceil(a / b * c) is equivalent to ceil(a * (c / b))
    aspect_x = size.x / size.y
    grid_x = int(math.ceil(voxels_on_y * aspect_x))

    # Calculate Z dimension: scale by aspect ratio z/y, round up
    aspect_z = size.z / size.y
    grid_z = int(math.ceil(voxels_on_y * aspect_z))

    return Vector3i(grid_x, grid_y, grid_z)
```

**Edge Cases and Pitfalls:**
- **Zero Height:** If `size.y` is zero, division by zero occurs. A guard clause should return a default or handle the degenerate case [G].
- **Precision:** Floating-point division followed by ceiling may introduce minor off-by-one errors if `size` components are very small or very large. Using integer arithmetic for aspect ratios is not possible here as `size` is continuous [G].
- **Non-Uniform Scaling:** This method assumes the grid cells are not necessarily cubic unless `size.x == size.y == size.z`. If cubic cells are required, a different method like [Grid Resolution Selection via Lambda](3270-grid-resolution-selection-via-lambda.md) should be used [S1, unofficial].

## Subtopics

### Adaptive Voxel Spacing
Adjusts voxel grid resolution dynamically using a quality parameter to balance performance and detail [S2, unofficial]. The quality parameter ranges from 0.0f to 3.0f, adjusted by key presses (+/-) [S2, unofficial]. The X and Y voxel sizes are calculated as `(2.0f * vw.aspx) / vw.xdim * (1.0f + quality * 4.0f)`, and the Z voxel size is calculated as `(2.0f * vw.aspz) / 200.0f * (1.0f + quality)` [S2, unofficial]. This method differs from the standard proportional calculation by allowing real-time user adjustment of voxel density via a scalar quality factor rather than a fixed geometric aspect ratio [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$, as it involves a constant number of arithmetic operations.
- **Memory Complexity:** $O(1)$, requiring only a few temporary variables.
- **Strengths:** Simple to implement; ensures the grid maintains the aspect ratio of the scene bounding box; easy to control resolution via a single Y-axis parameter.
- **Limitations:** Does not account for scene complexity (number of primitives); may result in non-cubic voxels if the bounding box is not a cube; does not adapt to local detail density.
- **When to use:** When a fixed aspect ratio is required and the scene bounding box is known and static.
- **When not to use:** When cubic voxels are required for optimal ray tracing performance or when the grid resolution needs to scale with the number of geometric primitives.

## Background
Originated in the context of voxel-based rendering and ray tracing optimizations, specifically in implementations like the `voxelizer` project [S1, unofficial]. It is used as a preliminary step in voxelization pipelines to determine the capacity of the voxel grid before occupancy counting or compression [S1, unofficial].

## Sources
[S1] loryruta/voxelizer, https://github.com/loryruta/voxelizer, fetched 2026-10-27 (unofficial)
[S2] jtomori/holographic_fractals, https://github.com/jtomori/holographic_fractals, fetched 2026-10-26 (unofficial)
