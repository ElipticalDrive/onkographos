# SDF Volume Generation & Ping-Pong Buffering

Also called: SDFGen, SDFMutate, double-buffered SDF update

## Overview
This method generates and interactively mutates a 3D Signed Distance Field (SDF) volume on the GPU using a ping-pong buffering scheme to handle concurrent read/write operations [S1, unofficial]. It solves the problem of maintaining geometric validity (the Eikonal equation) after local modifications by combining immediate ray-casting-based brush application with iterative Eikonal propagation [S1, unofficial]. It differs from static SDF generation by supporting real-time user input and from simple voxel carving by ensuring the resulting field remains a valid distance function through distance propagation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** `SDFResolution` (`Vector3Int`): Dimensions of the 3D SDF volume in voxels [S1, unofficial]. `SDFRadius` (`float`): Radius of the SDF brush/kernel used for mutation and Eikonal propagation [S1, unofficial]. `SDFMaxDist` (`float`): Maximum distance value to compute or store in the volume [S1, unofficial]. Mouse state: Origin (`_mouseOrigin`), Direction (`_mouseDir`), and flags for Add (`_mouseAdd`) or Subtract (`_mouseSubtract`) modes [S1, unofficial]. Brush parameters: `_brushSize`, `_brushHeight` [S1, unofficial].
*   **Outputs:** Updated SDF volume stored in the active ping-pong buffer [S1, unofficial].

**Data Structures:**
The SDF volume is stored in a ping-pong array of two 3D RenderTextures (`_sdfVolumeTexture[2]`) [S1, unofficial]. Each texture is configured with `TextureDimension.Tex3D`, `colorFormat = RenderTextureFormat.RFloat`, `enableRandomWrite = true`, `msaaSamples = 1`, `mipCount = 0`, and dimensions `width = SDFResolution.x`, `height = SDFResolution.y`, `volumeDepth = SDFResolution.z` [S1, unofficial]. Indices `_activeSdfTex` and `outSdfTex` track the current read and write buffers [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization:** Create two `RFloat` 3D render textures with `SDFResolution` [S1, unofficial]. Dispatch the `SDFGen` compute kernel to initialize the volume (e.g., with a base sphere) [S1, unofficial]. Calculate thread group counts: `_numKernels = SDFResolution / kernelSize` (where `kernelSize` is 8, matching `[numthreads(8,8,8)]`) [S1, unofficial].
2.  **Buffer Swapping:** On each update, calculate the next buffer index using bitwise operation: `outSdfTex = (_activeSdfTex + 1) & 1` [S1, unofficial]. Bind `_sdfVolumeTexture[inSdfTex]` as input and `_sdfVolumeTexture[outSdfTex]` as output [S1, unofficial]. Dispatch the `SDFMutate` kernel with `_numKernels` [S1, unofficial]. Update `_activeSdfTex = outSdfTex` [S1, unofficial].
3.  **Per-Voxel Mutation (SDFMutate Kernel):** For each voxel at `id`:
    a.  **Base SDF:** Calculate the distance to the base object (e.g., sphere) at world position `world` [S1, unofficial].
    b.  **Brush Application:** If mouse interaction is active: calculate intersection distance `toSdf` from mouse ray to base object [S1, unofficial]. Calculate ray origin `ro` at intersection [S1, unofficial]. Calculate distance `d` from voxel to brush center (`sdSphere(world, ro, brushSize)`) [S1, unofficial]. Apply Boolean operation: if `_mouseAdd`, `current = opSmoothUnion(d, current, 0.025)` [S1, unofficial]; if `_mouseSubtract`, `current = opSmoothSubtraction(d, current, 0.025)` [S1, unofficial].
    c.  **Eikonal Propagation:** If `_doEikonel` is true: fetch neighbor minimums $u_x = \min(|\text{sdf}(id + (1,0,0))|, |\text{sdf}(id - (1,0,0))|)$, $u_y = \min(|\text{sdf}(id + (0,1,0))|, |\text{sdf}(id - (0,1,0))|)$, $u_z = \min(|\text{sdf}(id + (0,0,1))|, |\text{sdf}(id - (0,0,1))|)$ [S1, unofficial]. Calculate grid step $g = \frac{\text{SDFRadius} \times 2.0}{\text{SDFResolution.x}}$ [S1, unofficial]. Compute Eikonal solution $e$ from $u = (u_x, u_y, u_z)$ and $g$ [S1, unofficial]. Update if current distance is larger than $g$: `if current > g: current = e` [S1, unofficial].
    d.  **Write:** Store `pack(current)` to output texture [S1, unofficial].
4.  **Propagation Passes (CPU-side):** If subtracting, calculate required Eikonal passes to fill the cut radius: $\text{passes} = \lfloor \frac{\text{BrushSize}}{\text{SDFRadius} \times 2.0 / \text{SDFResolution.x}} \rfloor \times 2$ [S1, unofficial]. Dispatch the mutation kernel repeatedly for this count [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def kernel_to_uv(id, resolution):
    return (id[0] / resolution[0], id[1] / resolution[1], id[2] / resolution[2])

def uv_to_world(uv):
    return (uv[0] * 2.0 - 1.0, uv[1] * 2.0 - 1.0, uv[2] * 2.0 - 1.0)

def sd_sphere(p, center, radius):
    return math.sqrt((p[0]-center[0])**2 + (p[1]-center[1])**2 + (p[2]-center[2])**2) - radius

def ray_sphere_intersect(origin, direction, center, radius):
    oc = (origin[0]-center[0], origin[1]-center[1], origin[2]-center[2])
    a = direction[0]**2 + direction[1]**2 + direction[2]**2
    b = 2.0 * (oc[0]*direction[0] + oc[1]*direction[1] + oc[2]*direction[2])
    c = oc[0]**2 + oc[1]**2 + oc[2]**2 - radius**2
    disc = b**2 - 4*a*c
    if disc < 0: return -1.0
    sqrt_disc = math.sqrt(disc)
    t1 = (-b - sqrt_disc) / (2*a)
    t2 = (-b + sqrt_disc) / (2*a)
    return t1 if t1 > 0 else t2 if t2 > 0 else -1.0

def op_smooth_union(d1, d2, k):
    h = max(0.0, min(1.0, 0.5 + 0.5 * (d2 - d1) / k))
    return d2 * h + d1 * (1.0 - h) - k * h * (1.0 - h)

def op_smooth_subtraction(d1, d2, k):
    h = max(0.0, min(1.0, 0.5 + 0.5 * (d2 - d1) / k))
    return d2 * h - d1 * (1.0 - h) + k * h * (1.0 - h)

def neighbor_min(tex, id, delta, tex_size):
    def idx(i, j, k):
        return ((i + j * tex_size[0]) * tex_size[1] + k)
    x, y, z = id
    v1 = tex[idx(x+delta[0], y+delta[1], z+delta[2])]
    v2 = tex[idx(x-delta[0], y-delta[1], z-delta[2])]
    return min(abs(v1), abs(v2))

def eikonal_1d(h, v, g):
    return min(h, v) + g

def eikonal_2d(h, v, g):
    if abs(h - v) >= g:
        return eikonal_1d(h, v, g)
    hv = h + v
    d = hv * hv - 2.0 * (h * h + v * v - g * g)
    return 0.5 * (hv + math.sqrt(d))

def eikonal_3d(u, g):
    xyz = u[0] + u[1] + u[2]
    n = 3.0
    d = (xyz * xyz) - n * ((u[0]**2 + u[1]**2 + u[2]**2) - (g * g))
    if d < 0.0:
        return None
    return (1.0 / n) * (xyz + math.sqrt(d))

def eikonal(u, g):
    e = eikonal_3d(u, g)
    if e is not None:
        return e
    e1 = eikonal_2d(u[0], u[1], g)
    e2 = eikonal_2d(u[0], u[2], g)
    e3 = eikonal_2d(u[1], u[2], g)
    return min(e1, e2, e3)

def pack(value):
    return value

def sdf_mutate_kernel(id, sdf_in, sdf_out, resolution, sdf_radius, brush_size, brush_height,
                      mouse_origin, mouse_dir, mouse_add, mouse_subtract, do_eikonal):
    uv = kernel_to_uv(id, resolution)
    world_pos = uv_to_world(uv)
    
    current = sd_sphere(world_pos, (0.0, 0.0, 0.0), 1.0)
    
    if mouse_add or mouse_subtract:
        to_sdf = ray_sphere_intersect(mouse_origin, mouse_dir, (0.0, 0.0, 0.0), 1.0)
        if to_sdf >= 0.0:
            ro = (mouse_origin[0] + mouse_dir[0]*to_sdf,
                  mouse_origin[1] + mouse_dir[1]*to_sdf,
                  mouse_origin[2] + mouse_dir[2]*to_sdf)
            d = sd_sphere(world_pos, ro, brush_size * (1.25 if mouse_subtract else 1.0))
            if mouse_add:
                current = op_smooth_union(d, current, 0.025)
            elif mouse_subtract:
                current = op_smooth_subtraction(d, current, 0.025)
                
    if do_eikonal:
        u = (neighbor_min(sdf_in, id, (1,0,0), resolution),
             neighbor_min(sdf_in, id, (0,1,0), resolution),
             neighbor_min(sdf_in, id, (0,0,1), resolution))
        g = sdf_radius * 2.0 / resolution[0]
        e = eikonal(u, g)
        if current > g:
            current = e
            
    sdf_out[id] = pack(current)
```

**Edge Cases and Pitfalls:**
*   **Precision:** Using `RFloat` (32-bit) is critical for distance accuracy. Lower precision formats will cause artifacts in Eikonal propagation [S1, unofficial].
*   **Boundary Conditions:** The `neighborMin` function accesses neighbors. The source assumes full access, implying the kernel size and dispatch cover the volume without out-of-bounds reads, or the texture handles wrapping/clamping [S1, unofficial].
*   **Eikonal Validity:** The Eikonal solution is only valid if the gradient magnitude is 1. The `eikonal` function approximates this. If `d < 0` in `eikonal3d`, it falls back to 2D solutions. This ensures stability but may reduce accuracy in complex topologies [S1, unofficial].
*   **Brush Size vs. Resolution:** If `BrushSize` is smaller than the voxel size, the brush may not be visible. The source scales the brush size for subtraction (`* 1.25f`) to compensate [S1, unofficial].

## Subtopics

### [SDF Mutation & Eikonal Propagation](4752-sdf-mutation-eikonal-propagation.md)
SDF Mutation & Eikonal Propagation is a GPU-based method for interactively modifying Signed Distance Fields (SDFs) using volumetric brushes. It solves the problem of maintaining geometrically correct distance values after local modifications (addition or subtraction) by combining immediate ray-casting-based brush application with iterative Eikonal equation propagation. It differs from static SDF generation by supporting real-time user input and from simple voxel carving by ensuring the resulting field remains a valid distance function through distance propagation.

## Operating characteristics
*   **Time Complexity:** $O(N)$ per voxel per pass, where $N$ is the number of voxels in the volume. The number of passes depends on the brush size and resolution [G].
*   **Memory Complexity:** $2 \times \text{SDFResolution.x} \times \text{SDFResolution.y} \times \text{SDFResolution.z} \times 4$ bytes (for two `RFloat` textures) [S1, unofficial].
*   **Performance:** The method is designed for real-time interaction on the GPU. Performance scales with resolution and the number of Eikonal passes [S1, unofficial].
*   **Strengths:** Real-time interactive modification, geometrically valid SDFs, efficient GPU implementation [S1, unofficial].
*   **Limitations:** Limited to GPU environments, requires careful tuning of brush size and resolution, Eikonal propagation is an approximation [S1, unofficial].
*   **When to Use:** Interactive SDF editing, real-time procedural geometry generation [S1, unofficial].
*   **When Not to Use:** Static SDF generation, CPU-only environments, high-precision offline rendering [G].

## Background
This method is derived from the implementation in the Unity project `samuelbigos/signed_distance_fields_in_unity` [S1, unofficial]. It leverages GPU compute shaders for parallel processing and ping-pong buffering to handle read/write conflicts. The Eikonal propagation is based on the fast marching method or similar distance transform algorithms adapted for GPU execution [G].

## Sources
[S1] samuelbigos/signed_distance_fields_in_unity, https://github.com/samuelbigos/signed_distance_fields_in_unity, fetched 2026-10-27 (unofficial)
