# Grid Resolution Calculation

Also called: proportional grid, voxel count

## Overview
Calculates a 3D grid resolution based on a target number of voxels on the Y-axis and the aspect ratio of the scene bounding box [S1, unofficial]. It solves the problem of determining uniform grid dimensions that preserve the scene's aspect ratio while fixing the vertical granularity. It is distinguished from [Grid Resolution Selection via Lambda](3270-grid-resolution-selection-via-lambda.md) by its explicit dependence on a fixed Y-axis voxel count rather than a primitive-count-based scaling factor $\lambda$.

## How it works
**Inputs:**
- `voxels_on_y`: Target number of voxels along the Y-axis (integer).
- `size`: Bounding box dimensions as a 3D vector $(size.x, size.y, size.z)$ [S1, unofficial].

**Outputs:**
- `grid_x`: Number of voxels along the X-axis.
- `grid_y`: Number of voxels along the Y-axis (equal to `voxels_on_y`).
- `grid_z`: Number of voxels along the Z-axis.

**Data Structures:**
- `size`: `vec3` representing the physical extent of the scene.
- `grid`: `uvec3` storing the resolution $(N_x, N_y, N_z)$.

**Steps:**
1. Set $N_y = \text{voxels\_on\_y}$.
2. Calculate $N_x = \lceil N_y \times \frac{size.x}{size.y} \rceil$.
3. Calculate $N_z = \lceil N_y \times \frac{size.z}{size.y} \rceil$.

**Formulas:**
$$ N_y = \text{voxels\_on\_y} $$
$$ N_x = \left\lceil \frac{\text{voxels\_on\_y}}{size.y} \times size.x \right\rceil $$
$$ N_z = \left\lceil \frac{\text{voxels\_on\_y}}{size.y} \times size.z \right\rceil $$

**Parameters:**
- `voxels_on_y`: User-defined or heuristic-derived integer. No typical value given in sources.

**Edge Cases and Pitfalls:**
- If `size.y` is zero or near-zero, division by zero occurs. [G] Guard against `size.y <= epsilon`.
- The use of `ceil` ensures the grid covers the full bounding box, potentially resulting in slightly smaller voxels than strictly proportional if the aspect ratio is not an integer multiple.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_grid_resolution(voxels_on_y: int, size: Vec3) -> Vec3:
    """
    Calculate grid resolution based on Y-axis voxel count and bounding box aspect ratio.
    
    Args:
        voxels_on_y: Target number of voxels along the Y-axis.
        size: Bounding box dimensions (x, y, z).
        
    Returns:
        Vec3: Grid resolution (Nx, Ny, Nz).
    """
    if size.y <= 1e-9:
        raise ValueError("Bounding box Y dimension must be positive.")
        
    ny = voxels_on_y
    nx = ceil(voxels_on_y * (size.x / size.y))
    nz = ceil(voxels_on_y * (size.z / size.y))
    
    return Vec3(nx, ny, nz)
```

## Subtopics

### Adaptive Voxel Spacing
Adjusts voxel grid resolution dynamically using a quality parameter to balance performance and detail [S2, unofficial]. It differs from the base method by allowing runtime adjustment of voxel size via a quality scalar, rather than a fixed Y-count [S2, unofficial].
- **Definition**: Dynamic voxel sizing based on a quality parameter [S2, unofficial].
- **Steps**:
  1. Set quality parameter $q \in [0.0, 3.0]$ [S2, unofficial].
  2. Calculate X/Y voxel size: $s_{xy} = \frac{2.0 \times vw.aspx}{vw.xdim} \times (1.0 + q \times 4.0)$ [S2, unofficial].
  3. Calculate Z voxel size: $s_z = \frac{2.0 \times vw.aspz}{200.0} \times (1.0 + q)$ [S2, unofficial].
- **When to choose**: When interactive adjustment of detail level is required [S2, unofficial].

## Operating characteristics
- **Time Complexity**: $O(1)$ arithmetic operations.
- **Memory Complexity**: $O(1)$ auxiliary storage.
- **Strengths**: Simple, deterministic, preserves aspect ratio.
- **Limitations**: Does not account for scene complexity (primitive count) or adaptivity. Fixed Y-count may lead to inefficient voxel counts for non-uniform scenes.
- **When to use**: When a fixed vertical resolution is desired for consistency across frames or scenes.
- **When not**: When memory is constrained and scene complexity varies significantly.

## Background
Originated in the `loryruta/voxelizer` project [S1, unofficial]. It provides a straightforward method for setting up uniform voxel grids for ray tracing or rendering by anchoring the resolution to a single axis.

## Sources
[S1] loryruta/voxelizer, https://github.com/loryruta/voxelizer, fetched 2026-10-27 (unofficial)
[S2] jtomori/holographic_fractals, https://github.com/jtomori/holographic_fractals, fetched 2026-10-26 (unofficial)
