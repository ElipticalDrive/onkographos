# SDF Volume Generation & Ping-Pong Buffering

Also called: SDFGen, SDFMutate, double-buffered SDF update

## Overview
This method generates and iteratively updates a 3D Signed Distance Field (SDF) volume on the GPU using a ping-pong buffering scheme. It solves the problem of maintaining a valid distance field after local geometric modifications (such as mouse-driven carving or addition) by combining immediate ray-casting-based brush application with iterative Eikonal equation propagation [S1, unofficial]. It differs from static SDF generation by supporting real-time user input and from simple voxel carving by ensuring the resulting field remains a valid distance function through distance propagation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D SDF volume stored in a ping-pong buffer, mouse position/direction, brush parameters (radius, height), and a boolean flag for Eikonal propagation.
*   **Output:** An updated 3D SDF volume written to the alternate buffer.

**Data Structures:**
The SDF volume is stored in a ping-pong array of two 3D RenderTextures (`_sdfVolumeTexture[2]`) [S1, unofficial].
*   **Format:** `RenderTextureFormat.RFloat` (single-channel 32-bit float) [S1, unofficial].
*   **Dimensions:** `width = SDFResolution.x`, `height = SDFResolution.y`, `volumeDepth = SDFResolution.z` [S1, unofficial].
*   **Properties:** `enableRandomWrite = true`, `msaaSamples = 1`, `mipCount = 0` [S1, unofficial].
*   **Indices:** `_activeSdfTex` tracks the current read buffer; `outSdfTex` is the write buffer [S1, unofficial].

**Parameters:**
*   `SDFResolution` (Vector3Int): Dimensions of the volume. Typically set as `DebugWindow.Instance.SdfResolution * 8` [S1, unofficial].
*   `SDFRadius` (float, `_sdfRadius`): Radius of the SDF brush/kernel [S1, unofficial].
*   `SDFMaxDist` (float, `_sdfMaxDist`): Maximum distance value to compute or store [S1, unofficial].
*   `_brushSize` (float): Radius of the interaction brush [S1, unofficial].
*   `_brushHeight` (float): Height of the interaction brush [S1, unofficial].
*   `_mouseOrigin` (float4): Ray origin from mouse interaction [S1, unofficial].
*   `_mouseDir` (float4): Ray direction from mouse interaction [S1, unofficial].
*   `_mouseSubtract` (bool): Mode flag for subtraction [S1, unofficial].
*   `_doEikonel` (bool): Flag to trigger Eikonal propagation [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization:** Create two `RFloat` 3D render textures with `SDFResolution`. Calculate thread group counts: `_numKernels = SDFResolution / kernelSize` (where `kernelSize` is 8, matching `[numthreads(8,8,8)]`) [S1, unofficial]. Dispatch the `SDFGen` kernel to initialize the volume (e.g., with a base sphere) [S1, unofficial].

2.  **Buffer Swapping:** On each update, determine the output buffer index using bitwise operation: `outSdfTex = (_activeSdfTex + 1) & 1` (Note: Source uses `& 2` in text but logic implies toggle; standard ping-pong uses `& 1` or `1 - active`. The source code snippet `outSdfTex = (_activeSdfTex + 1) & 2` appears to be a typo in the source description or specific to a non-standard indexing, but standard practice is `1 - active` or `^ 1`. However, the source explicitly states `& 2`. Given `& 2` on an index 0 or 1 results in 0, this is likely a source error. We follow the standard ping-pong logic described in the alias "double-buffered": `outSdfTex = 1 - _activeSdfTex` [G]). Bind `_sdfVolumeTexture[_activeSdfTex]` as input and `_sdfVolumeTexture[outSdfTex]` as output to the `SDFMutate` kernel [S1, unofficial]. Dispatch with `_numKernels` [S1, unofficial].

3.  **Update Active Index:** After dispatch, update `_activeSdfTex = outSdfTex` [S1, unofficial].

4.  **Per-Voxel Processing (SDFMutate Kernel):** For each voxel at `id`:
    a.  **Base SDF Calculation:** Convert voxel UV to world position `world = uvToWorld(uv)`. Calculate the base signed distance `current` to the primary geometry (e.g., a sphere) using `sdSphere(world, center, radius)` [S1, unofficial].
    b.  **Mouse Interaction (Add/Subtract):**
        i.  If mouse buttons are pressed, cast a ray from `_mouseOrigin` along `_mouseDir` to find intersection with the base geometry (`raySphereIntersect`) [S1, unofficial].
        ii. Calculate the ray origin `ro` at the intersection point [S1, unofficial].
        iii. Calculate the distance `d` from `ro` to the voxel `world` using `sdSphere(world, ro, brushSize)` (or `brushSize * 1.25f` for subtract) [S1, unofficial].
        iv. Combine `d` and `current` using smooth union (`opSmoothUnion`) for addition or smooth subtraction (`opSmoothSubtraction`) for subtraction, with a smoothness factor (e.g., 0.025) [S1, unofficial].
    c.  **Eikonal Propagation:**
        i.  If `_doEikonel` is true, fetch neighbor distances. `u.x = neighborMin(id, uint3(1, 0, 0))`, `u.y = neighborMin(id, uint3(0, 1, 0))`, `u.z = neighborMin(id, uint3(0, 0, 1))` [S1, unofficial].
        ii. `neighborMin` computes `min(abs(sdfUV(kernel + delta)), abs(sdfUV(kernel - delta)))` [S1, unofficial].
        iii. Calculate grid step `g = _sdfRadius * 2.0 / _sdfTexSizeX` [S1, unofficial].
        iv. Compute Eikonal solution `e = eikonal(u, g)` [S1, unofficial].
        v. If `current > g`, update `current = e` [S1, unofficial].
    d.  **Write Output:** Write `pack(current)` to `_sdfTexOut[id.xyz]` [S1, unofficial].

**Eikonal Formulas:**
The `eikonal` function solves the Eikonal equation $|\nabla d| = 1$ iteratively [S1, unofficial].
*   **1D:** `eikonal1d(h, v, g) = min(h, v) + g` [S1, unofficial].
*   **2D:** If `|h - v| >= g`, return `eikonal1d(h, v, g)`. Else, `e = 0.5 * (h + v + sqrt((h+v)^2 - 2(h^2 + v^2 - g^2)))` [S1, unofficial].
*   **3D:** Let `xyz = u.x + u.y + u.z`, `n = 3.0`. Let `d = xyz^2 - n * (u.x^2 + u.y^2 + u.z^2 - g^2)`. If `d < 0`, fallback to 2D solutions. Else, `e = (1/n) * (xyz + sqrt(d))` [S1, unofficial].
*   **Final 3D:** Return `min(e_3d, min(e_2d_xy, min(e_2d_xz, e_2d_yz)))` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for SDFMutate kernel execution per voxel
def sdf_mutate_kernel(id, sdf_volume_in, sdf_volume_out, params):
    # 1. Coordinate Conversion
    uv = kernel_to_uv(id)
    world = uv_to_world(uv)
    
    # 2. Base Geometry SDF
    current = sd_sphere(world, center=(0,0,0), radius=params.sphere_radius)
    
    # 3. Mouse Interaction
    if params.mouse_add or params.mouse_subtract:
        # Ray cast to find interaction point on base geometry
        to_sdf = ray_sphere_intersect(params.mouse_origin, params.mouse_dir, (0,0,0), params.sphere_radius + params.brush_height)
        ro = params.mouse_origin + params.mouse_dir * to_sdf
        
        # Brush distance
        brush_r = params.brush_size * (1.25 if params.mouse_subtract else 1.0)
        d = sd_sphere(world, ro, brush_r)
        
        # Combine with base
        if params.mouse_add:
            current = op_smooth_union(d, current, smoothness=0.025)
        elif params.mouse_subtract:
            current = op_smooth_subtraction(d, current, smoothness=0.025)
            
    # 4. Eikonal Propagation
    if params.do_eikonal:
        # Fetch neighbors (min of absolute values of +/- neighbors)
        u_x = min(abs(sdf_volume_in[id + (1,0,0)]), abs(sdf_volume_in[id - (1,0,0)]))
        u_y = min(abs(sdf_volume_in[id + (0,1,0)]), abs(sdf_volume_in[id - (0,1,0)]))
        u_z = min(abs(sdf_volume_in[id + (0,0,1)]), abs(sdf_volume_in[id - (0,0,1)]))
        
        g = params.sdf_radius * 2.0 / params.sdf_tex_size_x
        
        # Solve Eikonal
        e = solve_eikonal_3d(u_x, u_y, u_z, g)
        
        # Update if distance is large enough
        if current > g:
            current = e
            
    # 5. Write
    sdf_volume_out[id] = pack(current)

def solve_eikonal_3d(u_x, u_y, u_z, g):
    # Try 3D solution
    xyz = u_x + u_y + u_z
    n = 3.0
    d = (xyz * xyz) - n * ((u_x**2 + u_y**2 + u_z**2) - (g**2))
    
    if d >= 0:
        return (1.0 / n) * (xyz + sqrt(d))
    
    # Fallback to 2D solutions
    e_xy = solve_eikonal_2d(u_x, u_y, g)
    e_xz = solve_eikonal_2d(u_x, u_z, g)
    e_yz = solve_eikonal_2d(u_y, u_z, g)
    
    return min(e_xy, min(e_xz, e_yz))

def solve_eikonal_2d(h, v, g):
    if abs(h - v) >= g:
        return min(h, v) + g
    
    hv = h + v
    d = (hv * hv) - 2.0 * (h**2 + v**2 - g**2)
    return 0.5 * (hv + sqrt(d))
```

**Edge Cases and Pitfalls:**
*   **Precision:** The use of `RFloat` (32-bit) may introduce precision errors in large volumes or very fine details [S1, unofficial].
*   **Boundary Conditions:** Neighbor access at volume boundaries must handle out-of-bounds reads (typically clamping or assuming infinity) [G].
*   **Eikonal Passes:** The number of Eikonal passes required to fill a cut radius is calculated as `_eikonelPasses = (int)(BrushSize / (SDFRadius * 2.0 / SDFResolution.x)) * 2` [S1, unofficial]. Failure to run enough passes leaves invalid distance values [S1, unofficial].

## Subtopics

### [SDF Mutation & Eikonal Propagation](4752-sdf-mutation-eikonal-propagation.md)
GPU-based method for interactive SDF modification using volumetric brushes. Combines ray-casting brush application with iterative Eikonal propagation. Chosen for real-time user input.

### [SDF to Density Conversion](1282-sdf-to-density-conversion.md)
Converts continuous SDF to volumetric density $\sigma$. Chosen for rendering SDFs as volumes.

### [Constrained Jacobi Smoothing for SDFs](1583-constrained-jacobi-smoothing-for-sdfs.md)
Smoothing technique to ensure SDF validity (Eikonal equation) from binary volumes. Chosen for post-processing generated SDFs.

### [Script-to-Shader SDF Preprocessor](6349-script-to-shader-sdf-preprocessor.md)
Preprocessing tool for SDF scripts. Chosen for preparing SDF definitions for shader execution.

## Operating characteristics
*   **Time Complexity:** $O(N)$ per voxel per update, where $N$ is the number of voxels in the volume. Eikonal propagation requires multiple passes, scaling with the brush radius relative to voxel size [S1, unofficial].
*   **Memory Complexity:** $2 \times \text{SDFResolution}_x \times \text{SDFResolution}_y \times \text{SDFResolution}_z \times 4$ bytes (for two RFloat textures) [S1, unofficial].
*   **Performance:** Performance depends on GPU compute capability and volume resolution. The use of compute shaders allows for parallel processing [S1, unofficial].
*   **Strengths:** Real-time interactivity, valid distance field maintenance, GPU acceleration [S1, unofficial].
*   **Limitations:** Requires multiple passes for large modifications, precision limits of RFloat [S1, unofficial].
*   **When to Use:** Interactive SDF editing, real-time volume generation [S1, unofficial].
*   **When Not to Use:** Static SDF generation, high-precision offline rendering [G].

## Background
This method is implemented in Unity using C# and HLSL compute shaders. It is part of a project demonstrating signed distance fields in Unity [S1, unofficial]. The specific implementation details, including the ping-pong buffering and Eikonal propagation formulas, are derived from the source code provided in the repository [S1, unofficial].

## Sources
[S1] samuelbigos/signed_distance_fields_in_unity, https://github.com/samuelbigos/signed_distance_fields_in_unity, fetched 2026-10-27 (unofficial)
