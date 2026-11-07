# Grid Resolution Calculation

Also called: proportional grid, voxel count

## Overview
Grid Resolution Calculation determines the discrete dimensions $(N_x, N_y, N_z)$ of a uniform 3D voxel grid based on the physical extent of the scene and a target density or aspect ratio [S1, unofficial]. It solves the problem of mapping a continuous bounding box to a discrete array while preserving the scene's proportions, ensuring that the resulting voxels are roughly cubic to optimize subsequent traversal or rendering costs [S1, unofficial]. It differs from fixed-resolution approaches by scaling dynamically with the scene's bounding box dimensions, and from adaptive methods by producing a single global resolution rather than varying cell sizes [S1, unofficial][S2, unofficial].

## How it works
**Inputs:**
- `size`: A 3D vector $(size.x, size.y, size.z)$ representing the physical dimensions (width, height, depth) of the scene's bounding box [S1, unofficial].
- `voxels_on_y`: An integer scalar representing the desired number of voxels along the Y-axis (height) [S1, unofficial].

**Outputs:**
- `grid`: A 3D integer vector $(N_x, N_y, N_z)$ representing the number of voxels in each dimension [S1, unofficial].

**Data Structures:**
- `size`: `vec3` (float or double).
- `voxels_on_y`: `int` or `float`.
- `grid`: `uvec3` or `vec3` (integer components).

**Steps:**
1. Set the Y-dimension of the grid to the target voxel count:
   $$N_y = \text{voxels\_on\_y}$$
2. Calculate the X-dimension by scaling the Y-count by the aspect ratio of the bounding box width to height, rounding up to the nearest integer:
   $$N_x = \lceil N_y \times \frac{size.x}{size.y} \rceil$$
3. Calculate the Z-dimension by scaling the Y-count by the aspect ratio of the bounding box depth to height, rounding up to the nearest integer:
   $$N_z = \lceil N_y \times \frac{size.z}{size.y} \rceil$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_grid_resolution(size: Vec3, voxels_on_y: float) -> Vec3:
    """
    Calculate grid resolution based on bounding box size and target Y voxels.
    
    Args:
        size: Bounding box dimensions (x, y, z).
        voxels_on_y: Target number of voxels along the Y-axis.
        
    Returns:
        Grid dimensions (Nx, Ny, Nz).
    """
    # Ensure size.y is not zero to avoid division by zero
    if size.y == 0:
        raise ValueError("Bounding box height (size.y) cannot be zero.")
        
    ny = int(voxels_on_y)
    
    # Calculate Nx and Nz using ceiling to ensure full coverage
    # ceil(a / b) is equivalent to floor((a + b - 1) / b) for integers, 
    # but here we use float division and ceil.
    nx = int(math.ceil(ny * (size.x / size.y)))
    nz = int(math.ceil(ny * (size.z / size.y)))
    
    return Vec3(nx, ny, nz)
```

**Edge Cases and Pitfalls:**
- **Zero Dimensions:** If `size.y` is zero, the aspect ratio is undefined. A guard must check for this [G].
- **Rounding:** Using `ceil` ensures the grid fully contains the bounding box. Using `floor` or `round` might result in voxels being clipped or the grid being smaller than the scene [S1, unofficial].
- **Aspect Ratio Distortion:** If the bounding box is extremely elongated (e.g., a very thin plane), the resulting grid may have very high resolution in one dimension and low in others, potentially leading to inefficient memory usage or traversal [S1, unofficial].

## Subtopics

### Adaptive Voxel Spacing
This method adjusts voxel grid resolution dynamically using a quality parameter to balance performance and detail [S2, unofficial]. It differs from the proportional grid method by allowing runtime adjustment of voxel size based on a user-defined quality factor, rather than deriving a fixed resolution from scene statistics [S2, unofficial]. It is chosen when interactive adjustment of detail level is required [S2, unofficial].
- **Definition:** Adjusts voxel grid resolution dynamically using a quality parameter to balance performance and detail [S2, unofficial].
- **Steps:**
  1. Define a quality parameter ranging from 0.0f to 3.0f, adjustable by key presses (+/-) [S2, unofficial].
  2. Calculate X and Y voxel sizes:
     $$voxel\_size.x = \frac{2.0f \times vw.aspx}{vw.xdim} \times (1.0f + quality \times 4.0f)$$
     $$voxel\_size.y = \frac{2.0f \times vw.aspx}{vw.xdim} \times (1.0f + quality \times 4.0f)$$
     (Note: The source formula for Y is identical to X in the provided snippet, implying isotropic scaling in the XY plane relative to the viewport aspect [S2, unofficial].)
  3. Calculate Z voxel size:
     $$voxel\_size.z = \frac{2.0f \times vw.aspz}{200.0f} \times (1.0f + quality)$$
- **Parameters:** `quality` $\in [0.0, 3.0]$ [S2, unofficial]. `vw.aspx`, `vw.xdim`, `vw.aspz` are viewport/window specific constants [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ for the proportional grid method, as it involves a constant number of arithmetic operations [S1, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary space [S1, unofficial].
- **Strengths:** Simple to implement; ensures cubic voxels for uniform traversal costs; scales with scene size [S1, unofficial].
- **Limitations:** Does not account for local scene complexity; may produce inefficient grids for highly non-uniform scenes [S1, unofficial].
- **When to use:** When a uniform grid is required and scene statistics (bounding box) are known [S1, unofficial].
- **When not to use:** When adaptive resolution is needed or when the scene has significant empty space that should be skipped [S1, unofficial][S2, unofficial].

## Background
The proportional grid method originates from voxelizer implementations, specifically the `loryruta/voxelizer` project, which calculates grid resolution based on a target number of voxels on the Y-axis and the aspect ratio of the scene bounding box [S1, unofficial]. The adaptive variant is found in the `jtomori/holographic_fractals` project, where it is used for interactive fractal rendering [S2, unofficial].

## Sources
[S1] loryruta/voxelizer, https://github.com/loryruta/voxelizer, fetched 2026-10-27 (unofficial)
[S2] jtomori/holographic_fractals, https://github.com/jtomori/holographic_fractals, fetched 2026-10-26 (unofficial)
