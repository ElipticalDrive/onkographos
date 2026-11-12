# SDF Mutation & Eikonal Propagation

Also called: SDFMutate, brush carving, distance field update

## Overview
SDF Mutation & Eikonal Propagation is a GPU-based method for interactively modifying Signed Distance Fields (SDFs) using volumetric brushes [S1, unofficial]. It solves the problem of maintaining geometrically correct distance values after local modifications (addition or subtraction) by combining immediate ray-casting-based brush application with iterative Eikonal equation propagation [S1, unofficial]. It differs from static SDF generation by supporting real-time user input and from simple voxel carving by ensuring the resulting field remains a valid distance function through distance propagation [G].

## How it works
**Inputs and Outputs:**
- Input: Current SDF texture (`_sdfTexIn`), camera ray (origin `_mouseOrigin`, direction `_mouseDir`), brush parameters (`_brushSize`, `_brushHeight`), and interaction mode (`_mouseAdd`, `_mouseSubtract`) [S1, unofficial].
- Output: Updated SDF texture (`_sdfTexOut`) [S1, unofficial].

**Data Structures:**
- `_sdfTexIn`, `_sdfTexOut`: `RWTexture3D<float>` representing the 3D volume [S1, unofficial].
- `_mouseOrigin`, `_mouseDir`: `float4` (x,y,z,w) where xyz is the vector and w is unused or padding [S1, unofficial].
- `_brushSize`, `_brushHeight`: `float` defining the cylindrical brush dimensions [S1, unofficial].
- `_doEikonel`: `bool` flag indicating if Eikonal propagation is required, set true when `_eikonelPasses > 0` [S1, unofficial].
- `_eikonelPasses`: `int` counter for remaining propagation passes [S1, unofficial].

**Steps:**
1. **Ray Casting:** Cast a ray from the camera through the mouse position to determine the interaction point in world space [S1, unofficial].
2. **Flag Setting:** Set `_mouseAdd` or `_mouseSubtract` based on the current interaction mode (mode == 0 or 1) [S1, unofficial].
3. **Dispatch:** Dispatch the `SDFMutate` compute kernel with thread groups of size `[8,8,8]` [S1, unofficial].
4. **Kernel Execution:** The kernel reads from `_sdfTexIn`, applies the ray/brush mutation, and writes to `_sdfTexOut` [S1, unofficial].
5. **Eikonal Propagation (Optional):** If `_doEikonel` is true, the kernel propagates distance values outward [S1, unofficial]. Decrement `_eikonelPasses` after each dispatch and repeat until it reaches zero, ensuring the carved volume is fully filled with correct distance values [S1, unofficial].

**Formulas:**
The Eikonal solver uses 1D, 2D, and 3D cases to compute the new distance value $e$ for a voxel with gradient magnitude $g$ and neighbor distances $u_x, u_y, u_z$ [S1, unofficial].

1D case:
$$ e = \min(h, v) + g $$
where $h, v$ are the distances of the two neighbors along the axis [S1, unofficial].

2D case:
If $|h - v| \ge g$:
$$ e = \min(h, v) + g $$
Else:
$$ e = \frac{1}{2} \left( (h+v) + \sqrt{(h+v)^2 - 2(h^2 + v^2 - g^2)} \right) $$
[S1, unofficial]

3D case:
Let $xyz = u_x + u_y + u_z$ and $n=3$ [S1, unofficial].
If $d < 0$, the 3D solution is invalid, fall back to 2D cases [S1, unofficial].
Else:
$$ e = \frac{1}{n} \left( xyz + \sqrt{xyz^2 - n(u_x^2 + u_y^2 + u_z^2 - g^2)} \right) $$
[S1, unofficial]

The final Eikonal value for a voxel is the minimum of the 3D solution (if valid) and the minimum of the three 2D solutions (xy, xz, yz) [S1, unofficial].

**Parameters:**
- `_brushSize`, `_brushHeight`: Brush dimensions [S1, unofficial].
- `_eikonelPasses`: Number of propagation passes [S1, unofficial].
- `MAX_RAYMARCH_STEPS` (int): Maximum number of steps allowed during raymarching in the renderer. Passed to both compute and fragment shaders [S1, unofficial].
- Typical values: Not given in the sources.

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point precision errors in the Eikonal solver can cause artifacts. The 3D case check $d < 0$ handles degenerate cases [S1, unofficial].
- **Boundaries:** Voxel indices must be clamped to the texture bounds during neighbor access [G].
- **Degenerate Input:** If the ray does not intersect the volume, no mutation occurs [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def eikonal1d(h, v, g):
    return min(h, v) + g

def eikonal2d(h, v, g):
    if abs(h - v) >= g:
        return eikonal1d(h, v, g)
    hv = h + v
    d = hv * hv - 2.0 * (h * h + v * v - g * g)
    return 0.5 * (hv + math.sqrt(d))

def eikonal3d(u, g):
    xyz = u[0] + u[1] + u[2]
    n = 3.0
    d = (xyz * xyz) - n * ((u[0]**2 + u[1]**2 + u[2]**2) - (g * g))
    if d < 0.0:
        return None
    return (1.0 / n) * (xyz + math.sqrt(d))

def eikonal(u, g):
    e = eikonal3d(u, g)
    if e is not None:
        return e
    e1 = eikonal2d(u[0], u[1], g)
    e2 = eikonal2d(u[0], u[2], g)
    e3 = eikonal2d(u[1], u[2], g)
    return min(e1, e2, e3)

def clamp(val, min_val, max_val):
    return max(min_val, min(val, max_val))

def SDFMutateKernel(id_xyz, sdf_in, sdf_out, mouse_origin, mouse_dir, brush_size, mouse_subtract, mouse_add, do_eikonel, width, height, depth):
    # id_xyz: (x, y, z) thread index
    world_pos = (float(id_xyz[0]), float(id_xyz[1]), float(id_xyz[2]))
    input_val = sdf_in[id_xyz]
    new_val = input_val

    # Ray-brush intersection check
    v_to_ray = (world_pos[0] - mouse_origin[0], world_pos[1] - mouse_origin[1], world_pos[2] - mouse_origin[2])
    t = v_to_ray[0]*mouse_dir[0] + v_to_ray[1]*mouse_dir[1] + v_to_ray[2]*mouse_dir[2]
    closest = (mouse_origin[0] + t*mouse_dir[0], mouse_origin[1] + t*mouse_dir[1], mouse_origin[2] + t*mouse_dir[2])
    dist_to_ray = math.sqrt((world_pos[0]-closest[0])**2 + (world_pos[1]-closest[1])**2 + (world_pos[2]-closest[2])**2)

    if dist_to_ray < brush_size:
        dist_to_boundary = brush_size - dist_to_ray
        if mouse_subtract:
            new_val = dist_to_boundary
        elif mouse_add:
            new_val = -dist_to_boundary

    # Eikonal propagation if enabled
    if do_eikonel:
        nx = clamp(id_xyz[0] + 1, 0, width - 1)
        ny = clamp(id_xyz[1] + 1, 0, height - 1)
        nz = clamp(id_xyz[2] + 1, 0, depth - 1)
        u = (sdf_in[(nx, id_xyz[1], id_xyz[2])],
             sdf_in[(id_xyz[0], ny, id_xyz[2])],
             sdf_in[(id_xyz[0], id_xyz[1], nz)])
        g = 1.0  # Gradient magnitude for unit grid spacing [G]
        new_val = eikonal(u, g)

    sdf_out[id_xyz] = new_val
```

## Subtopics

### [Hierarchical Scalar Field Sculpting](10058-hierarchical-scalar-field-sculpting.md)
A method for interactive 3D modeling where the user modifies a scalar field representing material density using localized tools. It solves the trade-off between global resolution limits and interactive performance by using an adaptive hierarchical grid (ntree) that subdivides only where necessary. It differs from fixed-resolution sculpting by allowing local refinement to capture tool details, and from pure octree methods by using a fixed $n=4$ subdivision factor and direct subsampling of field values to ensure vertical independence between hierarchy levels.

## Operating characteristics
- **Time Complexity:** $O(N)$ per voxel per dispatch, where $N$ is the number of voxels in the volume. Eikonal propagation adds $P$ passes, where $P$ is `_eikonelPasses` [G].
- **Memory Complexity:** $O(N)$ for the two SDF textures (ping-pong buffering) [G].
- **Measured Performance:** Performance depends on the resolution of the SDF volume and the number of Eikonal passes. The use of compute shaders allows for parallel processing of voxels [G].
- **Strengths:** Real-time interactive modification, maintains valid distance field properties [G].
- **Limitations:** Limited to GPU execution, requires careful tuning of Eikonal passes for accuracy vs. performance [G].
- **When to use:** Interactive 3D modeling, real-time sculpting [G].
- **When not to use:** Offline rendering where pre-computed SDFs are sufficient, CPU-only environments [G].

## Background
Originated in the context of real-time 3D modeling in Unity, as demonstrated by the `samuelbigos/signed_distance_fields_in_unity` repository [S1, unofficial]. It leverages GPU compute shaders for efficient voxel manipulation and distance field correction [S1, unofficial].

## Sources
[S1] samuelbigos/signed_distance_fields_in_unity, https://github.com/samuelbigos/signed_distance_fields_in_unity, fetched 2026-10-27 (unofficial)
