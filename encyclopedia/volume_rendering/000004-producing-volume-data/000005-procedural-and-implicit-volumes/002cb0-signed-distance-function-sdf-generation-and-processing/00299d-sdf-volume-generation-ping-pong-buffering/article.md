# SDF Volume Generation & Ping-Pong Buffering

Also called: SDFGen, SDFMutate, double-buffered SDF update

## Overview
This method generates and interactively mutates a 3D Signed Distance Field (SDF) on the GPU using a ping-pong buffering scheme to maintain data consistency during concurrent read/write operations. It solves the problem of updating a volumetric scalar field in real-time while preserving the Eikonal equation constraint ($|\nabla d(\mathbf{x})| = 1$) after local geometric modifications. It differs from static SDF generation by supporting real-time user input via volumetric brushes and from simple voxel carving by ensuring the resulting field remains a valid distance function through iterative Eikonal propagation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D SDF volume stored in a ping-pong buffer of two `RFloat` 3D render textures. Mouse position, direction, and brush parameters.
*   **Output:** The updated SDF volume written to the active output texture.

**Data Structures:**
The SDF volume is stored in a ping-pong array of two 3D RenderTextures (`_sdfVolumeTexture[2]`). Each texture is configured with:
*   `TextureDimension.Tex3D`
*   `colorFormat = RenderTextureFormat.RFloat`
*   `enableRandomWrite = true`
*   `msaaSamples = 1`
*   `mipCount = 0`
*   Dimensions: `width = SDFResolution.x`, `height = SDFResolution.y`, `volumeDepth = SDFResolution.z` [S1, unofficial].

**Parameters:**
*   `SDFResolution` (Vector3Int): Dimensions of the 3D SDF volume in voxels. Typically set as `DebugWindow.Instance.SdfResolution * 8` [S1, unofficial].
*   `SDFRadius` (float): Radius of the SDF brush/kernel used for mutation and Eikonal propagation. Passed to shaders as `_sdfRadius` [S1, unofficial].
*   `SDFMaxDist` (float): Maximum distance value to compute or store in the volume. Passed to shaders as `_sdfMaxDist` [S1, unofficial].
*   `_brushSize` (float): Size of the volumetric brush [S1, unofficial].
*   `_brushHeight` (float): Height of the volumetric brush [S1, unofficial].
*   `_mouseOrigin` (float4): Origin of the mouse ray [S1, unofficial].
*   `_mouseDir` (float4): Direction of the mouse ray [S1, unofficial].
*   `_mouseSubtract` (bool): Flag for subtraction mode [S1, unofficial].
*   `_mouseAdd` (bool): Flag for addition mode [S1, unofficial].
*   `_doEikonel` (bool): Flag to enable Eikonal propagation [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization:** Create two `RFloat` 3D render textures with dimensions `SDFResolution`. Dispatch the `SDFGen` compute kernel to initialize the field (e.g., with a base sphere). Thread group counts are calculated as `_numKernels = SDFResolution / kernelSize`, where `kernelSize` is 8 (from `[numthreads(8,8,8)]`) [S1, unofficial].
2.  **Buffer Selection:** Determine the output buffer index using bitwise operation: `outSdfTex = (_activeSdfTex + 1) & 2`. This ensures the output buffer is different from the input buffer `_activeSdfTex` [S1, unofficial].
3.  **Mutation Kernel Dispatch:** Bind the input texture `_sdfVolumeTexture[inSdfTex]` and output texture `_sdfVolumeTexture[outSdfTex]` to the `SDFMutate` kernel. Dispatch the kernel with `_numKernels` [S1, unofficial].
4.  **Buffer Swap:** After dispatch, update `_activeSdfTex = outSdfTex` [S1, unofficial].
5.  **Eikonal Propagation (Optional):** If `_mouseSubtract` is active and mode is 1, calculate the required number of Eikonal propagation passes to fill the cut radius: `_eikonelPasses = (int)(BrushSize / (SDFRadius * 2.0 / SDFResolution.x)) * 2`. This ensures the distance field is corrected over the affected region [S1, unofficial].

**Pseudocode (Compute Shader Kernel `SDFMutate`):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
kernel_size = 8
sdf_radius = _sdfRadius
sdf_tex_size_x = _sdfTexSizeX
brush_size = _brushSize
brush_height = _brushHeight
mouse_origin = _mouseOrigin.xyz
mouse_dir = _mouseDir.xyz
mouse_subtract = _mouseSubtract
mouse_add = _mouseAdd
do_eikonal = _doEikonel

# Helper: Convert kernel ID to UV coordinates
def kernel_to_uv(kernel_id):
    # Implementation depends on specific mapping, assumed to be linear or standard 3D index to UV
    # Placeholder for standard 3D texture coordinate calculation
    return float3(kernel_id.x / sdf_tex_size_x, kernel_id.y / sdf_tex_size_y, kernel_id.z / sdf_tex_size_z)

# Helper: Convert UV to World coordinates
def uv_to_world(uv):
    # Placeholder for world space transformation
    return float3(uv.x, uv.y, uv.z)

# Helper: Signed Distance to Sphere
def sd_sphere(world_pos, center, radius):
    return length(world_pos - center) - radius

# Helper: Smooth Union
def op_smooth_union(a, b, k):
    h = clamp(0.5 + 0.5 * (b - a) / k, 0.0, 1.0)
    return a + h * (b - a)

# Helper: Smooth Subtraction
def op_smooth_subtraction(a, b, k):
    h = clamp(0.5 - 0.5 * (b - a) / k, 0.0, 1.0)
    return b - h * (b - a)

# Helper: Ray-Sphere Intersection
def ray_sphere_intersect(ro, rd, center, radius):
    oc = ro - center
    b = dot(oc, rd)
    c = dot(oc, oc) - radius * radius
    disc = b * b - c
    if disc < 0.0:
        return -1.0
    disc_sqrt = sqrt(disc)
    t1 = -b - disc_sqrt
    t2 = -b + disc_sqrt
    if t1 > 0.0:
        return t1
    if t2 > 0.0:
        return t2
    return -1.0

# Helper: Neighbor Min
def neighbor_min(kernel_id, delta):
    # Read from input texture
    uv_in = kernel_to_uv(kernel_id + delta)
    val_a = sdf_uv(uv_in)
    uv_out = kernel_to_uv(kernel_id - delta)
    val_b = sdf_uv(uv_out)
    return min(abs(val_a), abs(val_b))

# Helper: Eikonal 1D
def eikonal_1d(h, v, g):
    return min(h, v) + g

# Helper: Eikonal 2D
def eikonal_2d(h, v, g):
    if abs(h - v) >= g:
        return eikonal_1d(h, v, g)
    hv = h + v
    d = hv * hv - 2.0 * (h * h + v * v - g * g)
    return 0.5 * (hv + sqrt(d))

# Helper: Eikonal 3D
def eikonal_3d(u, g):
    xyz = u.x + u.y + u.z
    n = 3.0
    d = (xyz * xyz) - n * ((u.x * u.x + u.y * u.y + u.z * u.z) - (g * g))
    if d < 0.0:
        return False, 0.0
    e = (1.0 / n) * (xyz + sqrt(d))
    return True, e

# Helper: Eikonal
def eikonal(u, g):
    success, e = eikonal_3d(u, g)
    if success:
        return e
    e1 = eikonal_2d(u.x, u.y, g)
    e2 = eikonal_2d(u.x, u.z, g)
    e3 = eikonal_2d(u.y, u.z, g)
    return min(min(e1, e2), e3)

# Main Kernel
@compute_kernel
def SDFMutate(id: uint3):
    uv = kernel_to_uv(id)
    world = uv_to_world(uv)
    
    # Initial SDF value (e.g., base sphere)
    current = sd_sphere(world, float3(0.0, 0.0, 0.0), _sphereRadius)
    
    # Mouse Interaction
    if mouse_add or mouse_subtract:
        # Calculate intersection with base sphere to get ray origin on surface
        to_sdf = ray_sphere_intersect(mouse_origin, mouse_dir, float3(0.0, 0.0, 0.0), _sphereRadius + brush_height)
        if to_sdf > 0.0:
            ro = mouse_origin + mouse_dir * to_sdf
            # Calculate distance to brush center
            d = sd_sphere(world, ro, brush_size * (1.25 if mouse_subtract else 1.0))
            
            if mouse_add:
                current = op_smooth_union(d, current, 0.025)
            elif mouse_subtract:
                current = op_smooth_subtraction(d, current, 0.025)
    
    # Eikonal Propagation
    if do_eikonal:
        u = float3(
            neighbor_min(id, uint3(1, 0, 0)),
            neighbor_min(id, uint3(0, 1, 0)),
            neighbor_min(id, uint3(0, 0, 1))
        )
        g = sdf_radius * 2.0 / sdf_tex_size_x
        e = eikonal(u, g)
        if current > g:
            current = e
            
    # Write output
    _sdfTexOut[id.xyz] = pack(current)
```

**Edge Cases and Pitfalls:**
*   **Precision:** The use of `RFloat` (32-bit float) is critical for maintaining distance accuracy. Lower precision formats may introduce artifacts in the distance field [S1, unofficial].
*   **Boundary Conditions:** The `neighborMin` function assumes valid neighbors. Boundary checks must be handled by the texture sampling or explicit bounds checking to avoid reading out-of-bounds memory [S1, unofficial].
*   **Eikonal Validity:** The Eikonal propagation only corrects values greater than `g`. Values below `g` are assumed to be valid or already corrected [S1, unofficial].
*   **Brush Size vs. Resolution:** The number of Eikonal passes depends on `BrushSize` and `SDFResolution`. If the brush is large relative to the resolution, insufficient passes may leave artifacts [S1, unofficial].

## Subtopics

### [SDF Mutation & Eikonal Propagation](4752-sdf-mutation-eikonal-propagation.md)
A GPU-based method for interactively modifying SDFs using volumetric brushes. It differs from static SDF generation by supporting real-time user input and from simple voxel carving by ensuring the resulting field remains a valid distance function through distance propagation. Choose this when real-time interactive sculpting of SDFs is required.

## Operating characteristics
*   **Time Complexity:** $O(N)$ per voxel per kernel dispatch, where $N$ is the number of voxels in the volume. Eikonal propagation requires multiple passes, scaling with the brush size and resolution.
*   **Memory Complexity:** $2 \times \text{SDFResolution.x} \times \text{SDFResolution.y} \times \text{SDFResolution.z} \times 4$ bytes (for `RFloat`).
*   **Performance:** Performance is limited by memory bandwidth due to the large texture reads/writes. The ping-pong buffering avoids synchronization issues but doubles memory usage.
*   **Strengths:** Real-time interactivity, GPU-accelerated, maintains distance field validity.
*   **Limitations:** Memory-intensive for high resolutions, requires multiple passes for large modifications, precision limitations of floating-point textures.
*   **When to use:** Interactive SDF sculpting, real-time procedural geometry generation.
*   **When not to use:** Static SDF generation, high-precision offline rendering, memory-constrained environments.

## Background
This method is implemented in the project `samuelbigos/signed_distance_fields_in_unity`, which demonstrates the use of SDFs in Unity for procedural geometry and interaction [S1, unofficial]. It builds on the principles of Eikonal solvers and GPU-based volume rendering.

## Sources
[S1] samuelbigos/signed_distance_fields_in_unity, https://github.com/samuelbigos/signed_distance_fields_in_unity, fetched 2026-10-27 (unofficial)
