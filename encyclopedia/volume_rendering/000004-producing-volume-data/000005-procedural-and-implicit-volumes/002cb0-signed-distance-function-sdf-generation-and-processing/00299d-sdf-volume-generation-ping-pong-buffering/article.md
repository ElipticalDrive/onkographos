# SDF Volume Generation & Ping-Pong Buffering

Also called: SDFGen, SDFMutate, double-buffered SDF update

## Overview
This method generates and interactively mutates a 3D Signed Distance Field (SDF) volume on the GPU using a ping-pong buffering scheme to handle concurrent read/write operations. It solves the problem of maintaining geometric validity (the Eikonal equation) after local modifications by combining immediate ray-casting-based brush application with iterative Eikonal propagation. It differs from static SDF generation by supporting real-time user input and from simple voxel carving by ensuring the resulting field remains a valid distance function through distance propagation.

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `SDFResolution` (`Vector3Int`): Dimensions of the 3D SDF volume in voxels.
    *   `SDFRadius` (`float`): Radius of the SDF brush/kernel used for mutation and Eikonal propagation.
    *   `SDFMaxDist` (`float`): Maximum distance value to compute or store.
    *   Mouse state: Origin (`_mouseOrigin`), Direction (`_mouseDir`), and flags for Add (`_mouseAdd`) or Subtract (`_mouseSubtract`) modes.
    *   Brush parameters: `_brushSize`, `_brushHeight`.
*   **Outputs:**
    *   Updated SDF volume stored in the active ping-pong buffer.

**Data Structures:**
The SDF volume is stored in a ping-pong array of two 3D RenderTextures (`_sdfVolumeTexture[2]`).
*   **Format:** `RenderTextureFormat.RFloat` (single-channel 32-bit float).
*   **Dimensions:** `width = SDFResolution.x`, `height = SDFResolution.y`, `volumeDepth = SDFResolution.z`.
*   **Properties:** `TextureDimension.Tex3D`, `enableRandomWrite = true`, `msaaSamples = 1`, `mipCount = 0`.
*   **Indices:** `_activeSdfTex` tracks the current read buffer; `outSdfTex` tracks the write buffer.

**Algorithm Steps:**

1.  **Initialization:**
    Create two `RFloat` 3D render textures with `SDFResolution`. Dispatch the `SDFGen` compute kernel to initialize the volume (e.g., with a base sphere). Calculate thread group counts: `_numKernels = SDFResolution / kernelSize` (where `kernelSize` is typically 8, matching `[numthreads(8,8,8)]`).

2.  **Buffer Swapping:**
    On each update, calculate the next buffer index using bitwise operation:
    $$ outSdfTex = (\text{activeSdfTex} + 1) \ \& \ 2 $$
    Bind `_sdfVolumeTexture[inSdfTex]` as input and `_sdfVolumeTexture[outSdfTex]` as output. Dispatch the `SDFMutate` kernel. Update `_activeSdfTex = outSdfTex`.

3.  **Per-Voxel Mutation (SDFMutate Kernel):**
    For each voxel at `id`:
    a.  **Base SDF:** Calculate the distance to the base object (e.g., sphere) at world position `world`.
    b.  **Brush Application:** If mouse interaction is active:
        *   Calculate intersection distance `toSdf` from mouse ray to base object.
        *   Calculate ray origin `ro` at intersection.
        *   Calculate distance `d` from voxel to brush center (`sdSphere(world, ro, brushSize)`).
        *   Apply Boolean operation:
            *   If `_mouseAdd`: `current = opSmoothUnion(d, current, 0.025)`
            *   If `_mouseSubtract`: `current = opSmoothSubtraction(d, current, 0.025)`
    c.  **Eikonal Propagation:** If `_doEikonel` is true:
        *   Fetch neighbor minimums:
            $$ u_x = \min(|\text{sdf}(id + (1,0,0))|, |\text{sdf}(id - (1,0,0))|) $$
            $$ u_y = \min(|\text{sdf}(id + (0,1,0))|, |\text{sdf}(id - (0,1,0))|) $$
            $$ u_z = \min(|\text{sdf}(id + (0,0,1))|, |\text{sdf}(id - (0,0,1))|) $$
        *   Calculate grid step $g$:
            $$ g = \frac{\text{SDFRadius} \times 2.0}{\text{SDFResolution.x}} $$
        *   Compute Eikonal solution $e$ from $u = (u_x, u_y, u_z)$ and $g$.
        *   Update if current distance is larger than $g$:
            $$ \text{if } current > g: \ current = e $$
    d.  **Write:** Store `pack(current)` to output texture.

4.  **Propagation Passes (CPU-side):**
    If subtracting, calculate required Eikonal passes to fill the cut radius:
    $$ \text{passes} = \left\lfloor \frac{\text{BrushSize}}{\text{SDFRadius} \times 2.0 / \text{SDFResolution.x}} \right\rfloor \times 2 $$
    Dispatch the mutation kernel repeatedly for this count.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Compute Shader: SDFMutate
# Inputs: RWTexture3D<float> _sdfTexOut, Texture3D<float> _sdfTexIn
# Uniforms: float _sdfRadius, float _sdfMaxDist, float _brushSize, float _brushHeight
# Uniforms: float4 _mouseOrigin, float4 _mouseDir, bool _mouseAdd, bool _mouseSubtract, bool _doEikonel
# Uniforms: int _sdfTexSizeX, int3 SDFResolution

def kernelToUV(id):
    # Convert thread ID to normalized UV coordinates [0,1]
    return float3(id.x / SDFResolution.x, id.y / SDFResolution.y, id.z / SDFResolution.z)

def world(uv):
    # Map UV to world space (implementation depends on scene setup)
    # Assuming unit cube centered at origin for simplicity
    return (uv - 0.5) * 2.0

def sdSphere(p, center, radius):
    return length(p - center) - radius

def opSmoothUnion(d1, d2, k):
    # Smooth union approximation
    h = clamp(0.5 + 0.5 * (d2 - d1) / k, 0.0, 1.0)
    return lerp(d2, d1, h) - k * h * (1.0 - h)

def opSmoothSubtraction(d1, d2, k):
    # Smooth subtraction (union of d1 and -d2)
    h = clamp(0.5 + 0.5 * (d2 - d1) / k, 0.0, 1.0)
    return lerp(d2, -d1, h) + k * h * (1.0 - h)

def neighborMin(tex, id, delta):
    v1 = tex[id + delta]
    v2 = tex[id - delta]
    return min(abs(v1), abs(v2))

def eikonal1d(h, v, g):
    return min(h, v) + g

def eikonal2d(h, v, g):
    if abs(h - v) >= g:
        return eikonal1d(h, v, g)
    hv = h + v
    d = hv * hv - 2.0 * (h * h + v * v - g * g)
    return 0.5 * (hv + sqrt(d))

def eikonal3d(u, g):
    # u is float3 (ux, uy, uz)
    xyz = u.x + u.y + u.z
    n = 3.0
    d = (xyz * xyz) - n * ((u.x * u.x + u.y * u.y + u.z * u.z) - (g * g))
    if d < 0.0:
        return None
    return (1.0 / n) * (xyz + sqrt(d))

def eikonal(u, g):
    e = eikonal3d(u, g)
    if e is not None:
        return e
    e1 = eikonal2d(u.x, u.y, g)
    e2 = eikonal2d(u.x, u.z, g)
    e3 = eikonal2d(u.y, u.z, g)
    return min(min(e1, e2), e3)

[numthreads(8, 8, 8)]
void SDFMutate(uint3 id : SV_DispatchThreadID) : SV_Target
{
    float3 uv = kernelToUV(id);
    float3 worldPos = world(uv);
    
    # Base SDF (Sphere at origin)
    float current = sdSphere(worldPos, float3(0.0, 0.0, 0.0), 1.0);
    
    # Mouse Interaction
    if (_mouseAdd or _mouseSubtract):
        # Ray intersection with base sphere to find brush position
        # Simplified: assume ray hits sphere at _mouseOrigin + _mouseDir * t
        # In source, raySphereIntersect is used. Here we approximate or assume pre-calculated ro
        # Source uses: float toSdf = raySphereIntersect(...); float3 ro = ...
        # For self-contained pseudocode, we assume ro is provided or calculated externally
        # Let's assume ro is passed or calculated. 
        # Note: Source calculates ro based on mouse ray and base sphere intersection.
        # We will assume a helper function raySphereIntersect exists or is implemented.
        
        # Placeholder for ray-sphere intersection logic from source
        # float t = raySphereIntersect(_mouseOrigin.xyz, _mouseDir.xyz, float3(0,0,0), 1.0);
        # float3 ro = _mouseOrigin.xyz + _mouseDir.xyz * t;
        # In source, ro is used to calculate brush distance.
        # For this example, we assume ro is available.
        float3 ro = _mouseOrigin.xyz + _mouseDir.xyz * 1.0; # Simplified
        
        float d = sdSphere(worldPos, ro, _brushSize * (_mouseSubtract ? 1.25f : 1.0f));
        
        if (_mouseAdd):
            current = opSmoothUnion(d, current, 0.025);
        elif (_mouseSubtract):
            current = opSmoothSubtraction(d, current, 0.025);
            
    # Eikonal Propagation
    if (_doEikonel):
        float3 u;
        u.x = neighborMin(_sdfTexIn, id, uint3(1, 0, 0));
        u.y = neighborMin(_sdfTexIn, id, uint3(0, 1, 0));
        u.z = neighborMin(_sdfTexIn, id, uint3(0, 0, 1));
        
        float g = _sdfRadius * 2.0 / _sdfTexSizeX;
        float e = eikonal(u, g);
        
        if (current > g):
            current = e;
            
    _sdfTexOut[id.xyz] = current;
}
```

**Edge Cases and Pitfalls:**
*   **Precision:** Using `RFloat` (32-bit) is critical for distance accuracy. Lower precision formats will cause artifacts in Eikonal propagation.
*   **Boundary Conditions:** The `neighborMin` function accesses neighbors. Ensure boundary checks are handled if the volume is not padded. The source assumes full access, implying the kernel size and dispatch cover the volume without out-of-bounds reads, or the texture handles wrapping/clamping.
*   **Eikonal Validity:** The Eikonal solution is only valid if the gradient magnitude is 1. The `eikonal` function approximates this. If `d < 0` in `eikonal3d`, it falls back to 2D solutions. This ensures stability but may reduce accuracy in complex topologies.
*   **Brush Size vs. Resolution:** If `BrushSize` is smaller than the voxel size, the brush may not be visible. The source scales the brush size for subtraction (`* 1.25f`) to compensate.

## Subtopics

### [SDF Mutation & Eikonal Propagation](4752-sdf-mutation-eikonal-propagation.md)
SDF Mutation & Eikonal Propagation is a GPU-based method for interactively modifying Signed Distance Fields (SDFs) using volumetric brushes. It solves the problem of maintaining geometrically correct distance values after local modifications (addition or subtraction) by combining immediate ray-casting-based brush application with iterative Eikonal equation propagation. It differs from static SDF generation by supporting real-time user input and from simple voxel carving by ensuring the resulting field remains a valid distance function through distance propagation.

## Operating characteristics
*   **Time Complexity:** $O(N)$ per voxel per pass, where $N$ is the number of voxels in the volume. The number of passes depends on the brush size and resolution.
*   **Memory Complexity:** $2 \times \text{SDFResolution.x} \times \text{SDFResolution.y} \times \text{SDFResolution.z} \times 4$ bytes (for two `RFloat` textures).
*   **Performance:** The method is designed for real-time interaction on the GPU. Performance scales with resolution and the number of Eikonal passes.
*   **Strengths:** Real-time interactive modification, geometrically valid SDFs, efficient GPU implementation.
*   **Limitations:** Limited to GPU environments, requires careful tuning of brush size and resolution, Eikonal propagation is an approximation.
*   **When to Use:** Interactive SDF editing, real-time procedural geometry generation.
*   **When Not to Use:** Static SDF generation, CPU-only environments, high-precision offline rendering.

## Background
This method is derived from the implementation in the Unity project `samuelbigos/signed_distance_fields_in_unity` [S1, unofficial]. It leverages GPU compute shaders for parallel processing and ping-pong buffering to handle read/write conflicts. The Eikonal propagation is based on the fast marching method or similar distance transform algorithms adapted for GPU execution.

## Sources
[S1] samuelbigos/signed_distance_fields_in_unity, https://github.com/samuelbigos/signed_distance_fields_in_unity, fetched 2026-10-27 (unofficial)
