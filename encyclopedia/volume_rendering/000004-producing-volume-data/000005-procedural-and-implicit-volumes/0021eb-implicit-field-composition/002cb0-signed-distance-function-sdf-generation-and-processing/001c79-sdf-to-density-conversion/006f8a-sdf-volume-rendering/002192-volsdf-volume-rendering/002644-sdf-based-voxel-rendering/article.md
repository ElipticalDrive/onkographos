# SDF-based Voxel Rendering

## Overview
SDF-based Voxel Rendering determines voxel visibility by evaluating a Signed Distance Function (SDF) at each voxel center and drawing the voxel if the result is less than or equal to zero [S1, unofficial]. It differs from ray-marching volume rendering by using a discrete grid traversal with a binary occupancy test based on the implicit field, rather than continuous integration of density along rays [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: An SDF function evaluating a shape, a voxel grid defined by view bounds and step sizes, and a frame buffer context [S1, unofficial].
- Output: A rendered image where voxels satisfying the condition are colored [S1, unofficial].

**Data Structures:**
- `point3d`: A tuple or structure containing three floating-point fields `x`, `y`, `z` [S1, unofficial].
- `voxie_frame_t`: The rendering context or frame buffer object [S1, unofficial].
- `voxie_drawvox`: A function that writes a colored voxel to the frame buffer at specified coordinates [S1, unofficial].

**Numbered Steps:**
1. Initialize the voxel grid bounds (e.g., $-vw.aspx$ to $vw.aspx$) and compute voxel step sizes based on view dimensions and resolution [S1, unofficial].
2. Iterate through the 3D grid using nested loops for the $x$, $y$, and $z$ axes [S1, unofficial].
3. Evaluate the SDF for the current point $\mathbf{p} = (p.x, p.y, p.z)$ [S1, unofficial].
4. Apply the emission condition: $SDF(\mathbf{p}) \le 0$ [S1, unofficial].
5. If the condition is true, call the drawing function with the point coordinates and a specified color (e.g., `0x00ff00` or `0xffffff`) [S1, unofficial].

**Formulas:**
The source provides hardcoded implementations for specific shapes rather than general parametric formulas [S1, unofficial]:
- Sphere: `length3d(p) <= 0.3f` (radius $r = 0.3$) [S1, unofficial].
- Torus: `length2d((length2d((p.x, p.z)) - 0.2f, p.y)) <= 0.1f` (major radius $R = 0.2$, minor radius $r = 0.1$) [S1, unofficial].
- Box: `length3d(max3d(abs3d(p) - (0.3f, 0.4f, 0.1f), 0.0f)) <= 0.0f` (half-extents $\mathbf{b} = (0.3, 0.4, 0.1)$) [S1, unofficial].

**Parameters:**
- Voxel size: Computed dynamically from view bounds and resolution (e.g., `step_x = (2.0 * vw.aspx) / vw.xdim * 5.0`, `step_z = (2.0 * vw.aspz) / 200.0 * 2.0`) [S1, unofficial].
- Color: Hardcoded constants like `0x00ff00` (green) or `0xffffff` (white) [S1, unofficial].

**Edge Cases and Pitfalls:**
- Precision: Floating-point comparisons ($SDF \le 0$) may suffer from precision errors at the surface boundary [G].
- Boundaries: The loop bounds must strictly cover the volume of interest; otherwise, voxels may be missed or drawn outside the intended view [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_sdf_voxels(view_bounds, resolution, sdf_func, color, frame_buffer):
    # Compute voxel step sizes exactly as in the source
    step_x = (2.0 * view_bounds.max_x) / resolution.x * 5.0
    step_y = (2.0 * view_bounds.max_y) / resolution.x * 5.0
    step_z = (2.0 * view_bounds.max_z) / 200.0 * 2.0

    # Helper to write a voxel to the frame buffer
    def draw_voxel(p, col):
        frame_buffer[p] = col

    # Iterate through the 3D grid
    x = -view_bounds.max_x
    while x < view_bounds.max_x:
        y = -view_bounds.max_y
        while y < view_bounds.max_y:
            z = -view_bounds.max_z
            while z < view_bounds.max_z:
                if sdf_func((x, y, z)) <= 0.0:
                    draw_voxel((x, y, z), color)
                z += step_z
            y += step_y
        x += step_x
```
## Subtopics

### [Patch-based Color Blending](18150-patch-based-color-blending.md)
A technique to jointly evaluate local and contextual radiance consistency for more reliable color predictions by rendering colors of a patch of size $k \times k$ instead of a single pixel. It leverages the local surface assumption and homography transformation to efficiently derive colors from multiple input views. This method is chosen when regions have weak texture or changing intensity, as it produces more informative and consistent shape context by incorporating neighboring geometric information.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of voxels in the grid, as each voxel is visited once [S1, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary space if rendering directly to the frame buffer, or $O(N)$ if storing a voxel grid [G].
- **Strengths:** Simple implementation; efficient for solid objects; no need for ray marching or density integration [S1, unofficial].
- **Limitations:** Aliasing at surfaces; no transparency or subsurface scattering; resolution-dependent quality [G].
- **When to use:** Real-time visualization of solid implicit shapes; voxel-based collision detection [S1, unofficial].
- **When not to use:** Volumetric data with varying density; soft shadows; translucent materials [G].
## Background
Originating from early voxel rendering techniques in the "voxie" library ecosystem, this method leverages the mathematical properties of Signed Distance Functions to determine occupancy in a discrete grid [S1, unofficial].
## Sources
[S1] jtomori/holographic_fractals, https://github.com/jtomori/holographic_fractals, fetched 2026-10-26 (unofficial)
