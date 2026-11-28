# SDF Volume Generation & Ping-Pong Buffering

Also called: SDFGen, SDFMutate, double-buffered SDF update

## Overview
This method generates and maintains a 3D Signed Distance Field (SDF) volume on the GPU using a double-buffered ping-pong architecture, supporting both initial generation and interactive volumetric mutation [S1, unofficial]. It differs from static SDF generation by integrating immediate brush-based modifications with iterative Eikonal propagation to restore distance field validity, and from simple voxel carving by ensuring the resulting scalar field satisfies the Eikonal equation locally [G].

## How it works
**Inputs and Outputs:**
- **Input:** A 3D SDF volume stored in a ping-pong buffer of two textures, mouse/brush parameters (origin, direction, radius, height, mode), and the current active buffer index [S1, unofficial].
- **Output:** An updated 3D SDF volume written to the opposite buffer in the ping-pong pair [S1, unofficial].

**Data Structures:**
The SDF volume is stored in a ping-pong array of two 3D RenderTextures (`_sdfVolumeTexture[2]`) [S1, unofficial]. Each texture is configured with:
- `TextureDimension`: `Tex3D`
- `colorFormat`: `RFloat` (single-channel 32-bit float)
- `enableRandomWrite`: `true`
- `msaaSamples`: `1`
- `mipCount`: `0`
- Dimensions: `width = SDFResolution.x`, `height = SDFResolution.y`, `volumeDepth = SDFResolution.z` [S1, unofficial].

**Parameters:**
- `SDFResolution` (Vector3Int): Dimensions of the 3D SDF volume in voxels. Set dynamically as `DebugWindow.Instance.SdfResolution * 8` [S1, unofficial].
- `SDFRadius` (float): Radius of the SDF brush/kernel used for mutation and Eikonal propagation. Passed to shaders as `_sdfRadius` [S1, unofficial].
- `SDFMaxDist` (float): Maximum distance value to compute or store in the volume. Passed to shaders as `_sdfMaxDist` [S1, unofficial].
- `BrushSize` (float): Radius of the interactive brush [S1, unofficial].
- `BrushHeight` (float): Height parameter for the brush interaction [S1, unofficial].

**Numbered Steps:**
1. **Initialization:** Create two `RFloat` 3D render textures with the target resolution defined by `SDFResolution` [S1, unofficial]. Dispatch the `SDFGen` compute kernel to populate the initial SDF (e.g., a sphere) into the first buffer [S1, unofficial]. Calculate thread group counts as `_numKernels = SDFResolution / kernelSize`, where `kernelSize` is 8 (from `[numthreads(8,8,8)]`) [S1, unofficial].
2. **Buffer Selection:** Determine the output buffer index using bitwise operation: `outSdfTex = (_activeSdfTex + 1) & 1` (corrected from the source's `& 2` to correctly toggle between two buffers) [S1, unofficial]. Update `_activeSdfTex = outSdfTex` after dispatch [S1, unofficial].
3. **Mutation Dispatch:** Bind the input texture `_sdfVolumeTexture[inSdfTex]` and output texture `_sdfVolumeTexture[outSdfTex]` to the `SDFMutate` kernel [S1, unofficial]. Dispatch the kernel with `_numKernels` [S1, unofficial].
4. **Eikonal Passes (if needed):** If the left mouse button is pressed and mode is subtract (or add), calculate the required number of Eikonal propagation passes to fill the cut radius: `_eikonelPasses = (int)(BrushSize / (SDFRadius * 2.0 / SDFResolution.x)) * 2` [S1, unofficial]. Dispatch the `SDFMutate` kernel with `_doEikonel = true` for this many iterations [S1, unofficial].

**Pseudocode (Compute Shader Kernel `SDFMutate`):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Thread group size: 8x8x8
# Inputs: tex_in (RWTexture3D<float>), tex_out (RWTexture3D<float>)
# Uniforms: mouse_origin, mouse_dir, sphere_radius, brush_size, brush_height
#           mouse_add, mouse_subtract, do_eikonal, sdf_radius
#           tex_size_x, tex_size_y, tex_size_z

def kernel_to_uv(id_x, id_y, id_z):
    return (id_x / tex_size_x, id_y / tex_size_y, id_z / tex_size_z)

def uv_to_world(uv):
    return (uv[0] * 2.0 - 1.0, uv[1] * 2.0 - 1.0, uv[2] * 2.0 - 1.0)

def sd_sphere(world_pos, center, radius):
    return length(world_pos - center) - radius

def ray_sphere_intersect(origin, direction, center, radius):
    oc = (origin[0] - center[0], origin[1] - center[1], origin[2] - center[2])
    b = dot(oc, direction)
    c = dot(oc, oc) - radius * radius
    disc = b * b - c
    if disc < 0:
        return 1e10
    t = -b - sqrt(disc)
    return max(t, 0.0)

def op_smooth_union(d1, d2, k):
    h = clamp(0.5 + 0.5 * (d2 - d1) / k, 0.0, 1.0)
    return mix(d2, d1, h) - k * h * (1.0 - h)

def op_smooth_subtraction(d1, d2, k):
    return op_smooth_union(d1, -d2, k)

def neighbor_min(kernel_id, delta):
    a = abs(tex_in[kernel_id + delta])
    b = abs(tex_in[kernel_id - delta])
    return min(a, b)

def eikonal1d(h, v, g):
    return min(h, v) + g

def eikonal2d(h, v, g):
    if abs(h - v) >= g:
        return eikonal1d(h, v, g)
    hv = h + v
    d = hv * hv - 2.0 * (h * h + v * v - g * g)
    return 0.5 * (hv + sqrt(d))

def eikonal3d(u, g):
    xyz = u[0] + u[1] + u[2]
    n = 3.0
    d = (xyz * xyz) - n * ((u[0] * u[0] + u[1] * u[1] + u[2] * u[2]) - (g * g))
    if d < 0.0:
        return False, 0.0
    e = (1.0 / n) * (xyz + sqrt(d))
    return True, e

def eikonal(u, g):
    valid, e = eikonal3d(u, g)
    if valid:
        return e
    e1 = eikonal2d(u[0], u[1], g)
    e2 = eikonal2d(u[0], u[2], g)
    e3 = eikonal2d(u[1], u[2], g)
    return min(min(e1, e2), e3)

def pack(value):
    return value

def SDFMutate(id_x, id_y, id_z):
    uv = kernel_to_uv(id_x, id_y, id_z)
    world = uv_to_world(uv)
    
    current = sd_sphere(world, (0.0, 0.0, 0.0), sphere_radius)
    
    if mouse_add or mouse_subtract:
        to_sdf = ray_sphere_intersect(mouse_origin, mouse_dir, (0.0, 0.0, 0.0), sphere_radius + brush_height)
        ro = (mouse_origin[0] + mouse_dir[0] * to_sdf,
              mouse_origin[1] + mouse_dir[1] * to_sdf,
              mouse_origin[2] + mouse_dir[2] * to_sdf)
        d = sd_sphere(world, ro, brush_size * (1.25 if mouse_subtract else 1.0))
        
        if mouse_add:
            current = op_smooth_union(d, current, 0.025)
        elif mouse_subtract:
            current = op_smooth_subtraction(d, current, 0.025)
            
    if do_eikonal:
        u = (
            neighbor_min((id_x, id_y, id_z), (1, 0, 0)),
            neighbor_min((id_x, id_y, id_z), (0, 1, 0)),
            neighbor_min((id_x, id_y, id_z), (0, 0, 1))
        )
        g = sdf_radius * 2.0 / tex_size_x
        e = eikonal(u, g)
        if current > g:
            current = e
            
    tex_out[(id_x, id_y, id_z)] = pack(current)
```

**Edge Cases and Pitfalls:**
- **Precision:** The use of `RFloat` (32-bit float) may lead to precision issues for very large volumes or fine details [G].
- **Boundary Conditions:** The `neighbor_min` function accesses neighbors. At volume boundaries, these accesses may read uninitialized or zero values, potentially causing artifacts [G].
- **Bitwise Operation:** The source uses `outSdfTex = (_activeSdfTex + 1) & 2`. For a two-buffer system, this should be `& 1` to correctly toggle between indices 0 and 1 [S1, unofficial].
- **Eikonal Passes Calculation:** The formula `_eikonelPasses = (int)(BrushSize / (SDFRadius * 2.0 / SDFResolution.x)) * 2` assumes a specific relationship between brush size, SDF radius, and resolution [S1, unofficial].

## Subtopics

### [SDF Mutation & Eikonal Propagation](4752-sdf-mutation-eikonal-propagation.md)
SDF Mutation & Eikonal Propagation is a GPU-based method for interactively modifying Signed Distance Fields (SDFs) using volumetric brushes. It solves the problem of maintaining geometrically correct distance values after local modifications (addition or subtraction) by combining immediate ray-casting-based brush application with iterative Eikonal equation propagation. It differs from static SDF generation by supporting real-time user input and from simple voxel carving by ensuring the resulting field remains a valid distance function through distance propagation [G].

## Operating characteristics
- **Time Complexity:** O(N) per voxel per pass, where N is the number of voxels in the volume [G]. Eikonal propagation requires multiple passes, scaling with the brush radius and resolution [S1, unofficial].
- **Memory Complexity:** O(N) for the two ping-pong buffers [G].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Performance degrades with increased resolution and brush size due to more Eikonal passes [S1, unofficial].
- **Strengths:** Real-time interactive modification, GPU-accelerated, maintains distance field validity [G].
- **Limitations:** Limited to GPU execution, potential precision issues, boundary artifacts [G].
- **When to use:** Interactive SDF editing, real-time volume modification [G].
- **When not to use:** Static SDF generation, CPU-only environments, high-precision requirements [G].

## Background
This method originates from a Unity-based implementation documented in a GitHub repository, leveraging compute shaders for parallel processing [S1, unofficial]. It is designed for interactive SDF manipulation within the Unity engine [S1, unofficial].

## Sources
[S1] samuelbigos/signed_distance_fields_in_unity, https://github.com/samuelbigos/signed_distance_fields_in_unity, fetched 2026-10-27 (unofficial)
