# Grid Resolution Calculation

Also called: proportional grid, voxel count

## Overview
Calculates the dimensions of a uniform 3D voxel grid based on the aspect ratio of the scene's bounding box and a target resolution along one axis. It ensures the grid maintains the spatial proportions of the scene while fixing the computational density along a primary dimension. It differs from [Grid Resolution Selection via Lambda](3270-grid-resolution-selection-via-lambda.md) by relying on geometric aspect ratios rather than primitive counts and a scaling factor $\lambda$.

## How it works
**Inputs:**
- `voxels_on_y`: Target number of voxels along the Y-axis (integer).
- `size`: Bounding box dimensions as a 3D vector $(size.x, size.y, size.z)$ (float).

**Outputs:**
- `grid_res`: 3D integer vector $(grid.x, grid.y, grid.z)$ representing the number of voxels in each dimension.

**Data Structures:**
- `size`: `struct { float x, y, z; }`
- `grid_res`: `struct { int x, y, z; }`

**Steps:**
1. Set $grid.y = voxels\_on\_y$.
2. Calculate $grid.x = \lceil voxels\_on\_y \times \frac{size.x}{size.y} \rceil$.
3. Calculate $grid.z = \lceil voxels\_on\_y \times \frac{size.z}{size.y} \rceil$.

**Formulas:**
$$
grid.y = voxels\_on\_y
$$
$$
grid.x = \left\lceil voxels\_on\_y \cdot \frac{size.x}{size.y} \right\rceil
$$
$$
grid.z = \left\lceil voxels\_on\_y \cdot \frac{size.z}{size.y} \right\rceil
$$

**Parameters:**
- `voxels_on_y`: Typical values depend on desired detail; not specified in sources.
- `size`: Derived from scene bounding box.

**Edge Cases:**
- If $size.y = 0$, division by zero occurs. [G] Implementation should clamp or handle degenerate bounding boxes.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_grid_resolution(voxels_on_y: int, size: Vector3) -> Vector3i:
    """
    Calculate grid resolution based on Y-axis target and bounding box aspect ratio.
    
    Args:
        voxels_on_y: Target voxel count for the Y-axis.
        size: Bounding box dimensions (x, y, z).
        
    Returns:
        Grid resolution (x, y, z) as integers.
    """
    if size.y == 0:
        raise ValueError("Bounding box Y dimension cannot be zero.")
        
    grid_y = voxels_on_y
    grid_x = int(math.ceil(voxels_on_y * (size.x / size.y)))
    grid_z = int(math.ceil(voxels_on_y * (size.z / size.y)))
    
    return Vector3i(grid_x, grid_y, grid_z)
```

## Subtopics

### Adaptive Voxel Spacing
Adjusts voxel grid resolution dynamically using a quality parameter to balance performance and detail [S2, unofficial]. The quality parameter ranges from 0.0f to 3.0f, adjustable via key presses (+/-) [S2, unofficial]. X and Y voxel sizes are calculated as $(2.0f \times vw.aspx) / vw.xdim \times (1.0f + quality \times 4.0f)$, while Z voxel size is $(2.0f \times vw.aspz) / 200.0f \times (1.0f + quality)$ [S2, unofficial]. This method differs from fixed proportional grids by allowing real-time adjustment of voxel size based on user input or performance metrics [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ for calculation.
- **Memory Complexity:** $O(1)$ for storage of resolution parameters.
- **Strengths:** Simple to implement; maintains aspect ratio; easy to tune via Y-axis target.
- **Limitations:** Does not account for scene complexity (primitive count); uniform grid may waste memory in sparse regions.
- **When to use:** When aspect ratio preservation is critical and scene complexity is relatively uniform.
- **When not to use:** For scenes with high variance in detail density or when primitive count is the primary driver of memory constraints.

## Background
Originated in voxelizer implementations for real-time rendering, specifically in the `voxelizer` project by loryruta [S1, unofficial]. It is used in voxel-based rendering pipelines where uniform grid resolution is required for efficient traversal and shading.

## Sources
[S1] loryruta/voxelizer, https://github.com/loryruta/voxelizer, fetched 2026-10-27 (unofficial)
[S2] jtomori/holographic_fractals, https://github.com/jtomori/holographic_fractals, fetched 2026-10-26 (unofficial)
