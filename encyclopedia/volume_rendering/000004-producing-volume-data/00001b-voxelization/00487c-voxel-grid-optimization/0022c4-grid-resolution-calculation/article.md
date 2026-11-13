# Grid Resolution Calculation

Also called: proportional grid, voxel count

## Overview
This method determines the integer dimensions of a 3D voxel grid by fixing the resolution along one axis (typically the Y-axis) and calculating the other axes (X and Z) based on the aspect ratios of the scene's bounding box [S1, unofficial]. It solves the problem of maintaining geometric proportions in the voxel grid while allowing the user to control the total voxel count via a single parameter. It is distinguished from [Grid Resolution Selection via Lambda](3270-grid-resolution-selection-via-lambda.md) by its reliance on bounding box aspect ratios rather than primitive counts, and from [Voxel Grid Resolution Calculation](13832-voxel-grid-resolution-calculation.md) by its specific use of a target Y-resolution as the primary driver [S1, unofficial].

## How it works
**Inputs:**
- `voxels_on_y`: Target number of voxels along the Y-axis (integer).
- `size`: A 3D vector representing the dimensions of the scene's bounding box $(size.x, size.y, size.z)$ [S1, unofficial].

**Outputs:**
- `grid_x`: Integer width of the voxel grid.
- `grid_y`: Integer height of the voxel grid (equal to `voxels_on_y`).
- `grid_z`: Integer depth of the voxel grid.

**Data Structures:**
- `size`: `vec3` (float x, y, z).
- `voxels_on_y`: `int` or `float` (treated as scalar target).
- `grid_x`, `grid_y`, `grid_z`: `int`.

**Steps:**
1. Set the Y-axis resolution directly to the target value:
   $$grid\_y = voxels\_on\_y$$
2. Calculate the X-axis resolution by scaling the Y-resolution by the ratio of the bounding box width to height:
   $$grid\_x = \lceil voxels\_on\_y \times \frac{size.x}{size.y} \rceil$$
3. Calculate the Z-axis resolution by scaling the Y-resolution by the ratio of the bounding box depth to height:
   $$grid\_z = \lceil voxels\_on\_y \times \frac{size.z}{size.y} \rceil$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def calculate_grid_resolution(voxels_on_y: float, size_x: float, size_y: float, size_z: float):
    """
    Calculate 3D grid dimensions based on a target Y-resolution and bounding box aspect ratios.
    
    Args:
        voxels_on_y: Target number of voxels along the Y-axis.
        size_x: Width of the scene bounding box.
        size_y: Height of the scene bounding box.
        size_z: Depth of the scene bounding box.
        
    Returns:
        tuple: (grid_x, grid_y, grid_z) as integers.
    """
    # Guard against division by zero
    if size_y == 0:
        raise ValueError("Bounding box height (size_y) cannot be zero.")
        
    # Calculate X and Z dimensions based on aspect ratios
    # Using ceiling to ensure the grid covers the entire bounding box
    grid_x = math.ceil(voxels_on_y * (size_x / size_y))
    grid_y = int(voxels_on_y)
    grid_z = math.ceil(voxels_on_y * (size_z / size_y))
    
    return (grid_x, grid_y, grid_z)
```

**Parameters:**
- `voxels_on_y`: Controls the granularity. Typical values depend on the desired visual fidelity and memory budget. Not given in the sources.
- `size`: Derived from the scene geometry's bounding box.

**Edge Cases and Pitfalls:**
- **Zero Height:** If `size.y` is zero, division by zero occurs. A guard is required [G].
- **Non-Uniform Scaling:** The method assumes the bounding box axes align with the grid axes. If the bounding box is rotated relative to the grid, the aspect ratios must be calculated based on the projected extents or the method may produce distorted grids [G].
- **Precision:** Using floating-point arithmetic for the ratios is standard. The `ceil` function ensures integer results that are at least as large as the proportional requirement.

## Subtopics

### Adaptive Voxel Spacing
This variant adjusts voxel grid resolution dynamically using a quality parameter to balance performance and detail [S2, unofficial]. It differs from the base method by allowing real-time adjustment of voxel sizes in X, Y, and Z directions independently based on a scalar quality value, rather than fixing one axis and deriving others from aspect ratios. It is chosen when interactive adjustment of detail is required.
- **Quality Parameter:** Ranges from 0.0f to 3.0f, adjustable via key presses (+/-) [S2, unofficial].
- **Voxel Size Calculation:**
  - X/Y voxel size: $(2.0f \times vw.aspx) / vw.xdim \times (1.0f + quality \times 4.0f)$
  - Z voxel size: $(2.0f \times vw.aspz) / 200.0f \times (1.0f + quality)$
  Where `vw.aspx`, `vw.xdim`, `vw.aspz` are scene/viewer specific constants [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ for the calculation itself, as it involves a constant number of arithmetic operations.
- **Memory Complexity:** $O(1)$ auxiliary space.
- **Strengths:** Simple to implement; ensures the voxel grid maintains the aspect ratio of the scene; easy to control total voxel count by adjusting the Y-axis target.
- **Limitations:** Does not account for the distribution of geometry within the bounding box (e.g., sparse regions); may result in wasted voxels if the bounding box contains empty space.
- **When to use:** When a uniform grid is desired and the scene's bounding box is well-defined and aligned with the grid axes.
- **When not to use:** When adaptive resolution is needed or when the scene geometry is sparse and irregularly distributed.

## Background
This method is implemented in the `voxelizer` project by loryruta [S1, unofficial]. It is used in voxel-based rendering pipelines where a uniform grid is required for efficient traversal or rendering, such as in ray marching or voxel cone tracing. The approach is a common heuristic in voxelization to avoid arbitrary grid dimensions while maintaining a consistent level of detail along a primary axis.

## Sources
[S1] loryruta/voxelizer, https://github.com/loryruta/voxelizer, fetched 2026-10-27 (unofficial)
[S2] jtomori/holographic_fractals, https://github.com/jtomori/holographic_fractals, fetched 2026-10-26 (unofficial)
