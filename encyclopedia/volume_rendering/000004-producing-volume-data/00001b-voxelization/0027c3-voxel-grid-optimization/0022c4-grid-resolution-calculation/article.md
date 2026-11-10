# Grid Resolution Calculation

Also called: proportional grid, voxel count

## Overview
This method determines the discrete dimensions $(N_x, N_y, N_z)$ of a uniform 3D voxel grid by anchoring the resolution on the Y-axis and scaling the X and Z axes according to the aspect ratios of the scene's spatial domain [S1, unofficial]. It solves the problem of maintaining geometric fidelity across non-cubic bounding boxes without requiring manual tuning of all three dimensions, ensuring the voxel density remains consistent with the scene's proportions. It differs from [Grid Resolution Selection via Lambda](3270-grid-resolution-selection-via-lambda.md) by relying solely on spatial aspect ratios rather than primitive counts, and from [Adaptive Voxel Spacing](3270-grid-resolution-selection-via-lambda.md#adaptive-voxel-spacing) by producing a fixed uniform grid rather than a quality-dependent variable spacing.

## How it works
**Inputs:**
- `voxels_on_y`: Target number of voxels along the Y-axis (integer).
- `size`: A 3D vector $(size.x, size.y, size.z)$ representing the physical dimensions (e.g., meters or world units) of the scene's bounding box [S1, unofficial].

**Outputs:**
- `resolution`: A 3D vector $(N_x, N_y, N_z)$ representing the number of voxels in each dimension [S1, unofficial].

**Data Structures:**
- `size`: `vec3` (float32).
- `resolution`: `uvec3` (uint32).

**Steps:**
1. Set the Y-resolution directly to the target value:
   $$N_y = \text{voxels\_on\_y}$$
2. Calculate the X-resolution by scaling the Y-target by the ratio of the scene's X-size to Y-size, rounding up to the nearest integer:
   $$N_x = \lceil \frac{\text{voxels\_on\_y}}{size.y} \times size.x \rceil$$
3. Calculate the Z-resolution by scaling the Y-target by the ratio of the scene's Z-size to Y-size, rounding up to the nearest integer:
   $$N_z = \lceil \frac{\text{voxels\_on\_y}}{size.y} \times size.z \rceil$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_grid_resolution(voxels_on_y: int, size: tuple[float, float, float]) -> tuple[int, int, int]:
    """
    Calculate uniform grid resolution based on Y-axis target and scene bounding box size.
    
    Args:
        voxels_on_y: Target number of voxels along the Y-axis.
        size: Tuple (x, y, z) representing the physical dimensions of the scene bounding box.
        
    Returns:
        Tuple (Nx, Ny, Nz) representing the grid resolution.
    """
    # Step 1: Y resolution is fixed by the target
    ny = voxels_on_y
    
    # Step 2: X resolution scales with aspect ratio x/y
    # Using math.ceil to ensure the grid covers the full extent
    nx = int((voxels_on_y / size[1]) * size[0])
    # Handle potential float-to-int truncation if not already ceiling
    if (voxels_on_y / size[1]) * size[0] > nx:
        nx += 1
        
    # Step 3: Z resolution scales with aspect ratio z/y
    nz = int((voxels_on_y / size[1]) * size[2])
    if (voxels_on_y / size[1]) * size[2] > nz:
        nz += 1
        
    return (nx, ny, nz)
```

**Parameters:**
- `voxels_on_y`: Typical values depend on desired detail; higher values increase memory usage linearly. Not given in the sources.
- `size`: Determined by the scene's axis-aligned bounding box (AABB).

**Edge Cases and Pitfalls:**
- **Zero Size:** If `size.y` is zero, division by zero occurs. A guard is required to handle degenerate bounding boxes [G].
- **Precision:** Floating-point division followed by ceiling ensures the grid bounds encompass the entire scene. Using floor or truncation may result in voxels being clipped at the boundaries [G].

## Subtopics

### Adaptive Voxel Spacing
This variant adjusts voxel grid resolution dynamically using a quality parameter to balance performance and detail [S2, unofficial]. It differs from the standard proportional calculation by allowing real-time adjustment of voxel size based on user input or scene complexity, rather than a fixed aspect-ratio derivation. It is chosen when interactive exploration or variable detail levels are required.
- **Quality Parameter:** Ranges from 0.0f to 3.0f, adjustable via key presses (+/-) [S2, unofficial].
- **X/Y Voxel Size Formula:**
  $$\text{voxel\_size}_{x/y} = \frac{2.0f \times vw.aspx}{vw.xdim} \times (1.0f + \text{quality} \times 4.0f)$$
- **Z Voxel Size Formula:**
  $$\text{voxel\_size}_z = \frac{2.0f \times vw.aspz}{200.0f} \times (1.0f + \text{quality})$$
- **Context:** Used in holographic fractal rendering where `vw.aspx`, `vw.xdim`, and `vw.aspz` are view-specific scaling factors [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ arithmetic operations. The calculation is instantaneous regardless of scene size.
- **Memory Complexity:** $O(1)$ auxiliary memory. The output is a small fixed-size vector.
- **Strengths:** Simple to implement; guarantees uniform voxel density relative to the scene's geometry; easy to tune via a single Y-axis parameter.
- **Limitations:** Does not account for local geometric complexity; may waste voxels in empty regions if the bounding box is loose; fixed uniform spacing cannot adapt to high-frequency details in specific areas.
- **When to use:** When a uniform grid is required and the scene bounding box is well-defined and tight.
- **When not to use:** For scenes with highly varying density where sparse structures or adaptive methods are preferred.

## Background
This method originates from the `voxelizer` project by loryruta, which implements a voxel-based rendering pipeline [S1, unofficial]. It is used in real-time voxel ray tracing applications where a predictable grid structure is needed for shader-based traversal. The approach is a standard technique in voxelization to ensure that the grid resolution matches the aspect ratio of the input geometry, preventing distortion in the rendered output.

## Sources
[S1] loryruta/voxelizer, https://github.com/loryruta/voxelizer, fetched 2026-10-27 (unofficial)
[S2] jtomori/holographic_fractals, https://github.com/jtomori/holographic_fractals, fetched 2026-10-26 (unofficial)
