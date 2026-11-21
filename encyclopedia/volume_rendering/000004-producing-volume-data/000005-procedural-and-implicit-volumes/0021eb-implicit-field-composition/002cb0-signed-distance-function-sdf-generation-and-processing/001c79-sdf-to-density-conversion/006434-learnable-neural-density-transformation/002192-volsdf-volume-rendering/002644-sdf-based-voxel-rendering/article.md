# SDF-based Voxel Rendering

Also called: SDF Voxelization, Implicit Voxel Rendering

## Overview
SDF-based Voxel Rendering determines voxel visibility by evaluating a Signed Distance Function (SDF) at each voxel center; a voxel is drawn if the SDF value is less than or equal to zero [S1, unofficial]. It differs from ray-marching volume rendering by using a discrete grid traversal with a binary occupancy test based on the implicit field, rather than continuous integration of density along rays [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A Signed Distance Function $SDF(\mathbf{p})$ defining a shape, a voxel grid defined by bounds and step sizes, and a frame buffer.
- Output: A rendered image where voxels satisfying the condition are colored.

**Data Structures:**
- `point3d`: A structure or tuple containing three floating-point fields: `x`, `y`, `z`.
- `voxie_frame_t`: The rendering context/frame buffer object.
- `voxie_drawvox`: A function signature `void voxie_drawvox(voxie_frame_t *vf, float x, float y, float z, uint32_t color)`.

**Numbered Steps:**
1. Initialize the voxel grid bounds (e.g., $-vw.aspx$ to $vw.aspx$) and compute voxel step sizes based on resolution and field of view [S1, unofficial].
2. Iterate through the 3D grid using nested loops for $x$, $y$, and $z$ axes [S1, unofficial]:
   $$ \text{for } p.x \text{ from } -vw.aspx \text{ to } vw.aspx \text{ step } \Delta x $$
   $$ \text{for } p.y \text{ from } -vw.aspy \text{ to } vw.aspy \text{ step } \Delta y $$
   $$ \text{for } p.z \text{ from } -vw.aspz \text{ to } vw.aspz \text{ step } \Delta z $$
3. Evaluate the SDF for the current point $\mathbf{p} = (p.x, p.y, p.z)$ [S1, unofficial].
4. Apply the emission condition:
   $$ SDF(\mathbf{p}) \le 0 $$
5. If the condition is true, call the drawing function with the point coordinates and a specified color (e.g., `0x00ff00` or `0xffffff`) [S1, unofficial].

**Formulas:**
- **Sphere SDF:** $SDF(\mathbf{p}) = \|\mathbf{p}\| - r$ [S1, unofficial].
- **Torus SDF:** Let $\mathbf{t} = (R, r)$ be major and minor radii. Let $\mathbf{q} = (\|\mathbf{p}_{xz}\| - R, p_y)$. Then $SDF(\mathbf{p}) = \|\mathbf{q}\| - r$ [S1, unofficial].
- **Box SDF:** Let $\mathbf{b}$ be half-extents. Let $\mathbf{q} = \max(\|\mathbf{p}\| - \mathbf{b}, \mathbf{0})$. Then $SDF(\mathbf{p}) = \|\mathbf{q}\| + \max(\min(\|\mathbf{p}\| - \mathbf{b}), 0)$ [S1, unofficial].

**Parameters:**
- Voxel size: Computed dynamically based on view volume dimensions and resolution (e.g., `voxel_size.x = (2.0f * vw.aspx) / vw.xdim * 5.0f`) [S1, unofficial].
- Color: Hardcoded constants like `0x00ff00` (green) or `0xffffff` (white) [S1, unofficial].

**Edge Cases and Pitfalls:**
- Precision: Floating-point comparisons ($SDF \le 0$) may suffer from precision errors at the surface boundary [G].
- Boundaries: The loop bounds must strictly cover the volume of interest; otherwise, voxels may be missed or drawn outside the intended view [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_sdf_voxels(view_bounds, resolution, sdf_func, color):
    # Compute voxel step sizes
    step_x = (2.0 * view_bounds.max_x) / resolution.x
    step_y = (2.0 * view_bounds.max_y) / resolution.y
    step_z = (2.0 * view_bounds.max_z) / resolution.z
    
    # Iterate through grid
    for x in range(-view_bounds.max_x, view_bounds.max_x, step_x):
        for y in range(-view_bounds.max_y, view_bounds.max_y, step_y):
            for z in range(-view_bounds.max_z, view_bounds.max_z, step_z):
                p = Point3D(x, y, z)
                
                # Evaluate SDF
                dist = sdf_func(p)
                
                # Check visibility condition
                if dist <= 0.0:
                    draw_voxel(p.x, p.y, p.z, color)
```

## Subtopics

### [Patch-based Color Blending](18150-patch-based-color-blending.md)
A technique to jointly evaluate local and contextual radiance consistency for more reliable color predictions by rendering colors of a patch of size k x k instead of a single pixel.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of voxels in the grid, as each voxel is visited once [S1, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary space if rendering directly to the frame buffer, or $O(N)$ if storing a voxel grid [G].
- **Strengths:** Simple implementation; efficient for solid objects; no need for ray marching or density integration [S1, unofficial].
- **Limitations:** Aliasing at surfaces; no transparency or subsurface scattering; resolution-dependent quality [G].
- **When to use:** Real-time visualization of solid implicit shapes; voxel-based collision detection [S1, unofficial].
- **When not to use:** Volumetric data with varying density; soft shadows; translucent materials [G].

## Background
Originating from early voxel rendering techniques in the "voxie" library ecosystem, this method leverages the mathematical properties of Signed Distance Functions to determine occupancy in a discrete grid [S1, unofficial]. It is commonly used in procedural fractal rendering and holographic displays where solid implicit shapes are rendered efficiently [S1, unofficial].

## Sources
[S1] jtomori/holographic_fractals, https://github.com/jtomori/holographic_fractals, fetched 2026-10-26 (unofficial)
