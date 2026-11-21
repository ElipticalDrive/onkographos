# SDF Volume Generation & Ping-Pong Buffering

Also called: SDFGen, SDFMutate, double-buffered SDF update

## Overview
This method generates and maintains a 3D Signed Distance Field (SDF) volume on the GPU using a double-buffered ping-pong architecture, supporting both initial generation and interactive volumetric mutation [S1, unofficial]. It differs from static SDF generation by integrating immediate brush-based modifications with iterative Eikonal propagation to restore distance field validity, and from simple voxel carving by ensuring the resulting scalar field satisfies the Eikonal equation locally [G].

## How it works
**Inputs and Outputs:**
- **Input:** A 3D SDF volume stored in a ping-pong buffer of two textures, mouse/brush parameters (origin, direction, radius, height, mode), and the current active buffer index.
- **Output:** An updated 3D SDF volume written to the opposite buffer in the ping-pong pair.

**Data Structures:**
The SDF volume is stored in a ping-pong array of two 3D RenderTextures (`_sdfVolumeTexture[2]`) [S1, unofficial]. Each texture is configured with:
- `TextureDimension`: `Tex3D`
- `colorFormat`: `RFloat` (single-channel 32-bit float)
- `enableRandomWrite`: `true`
- `msaaSamples`: `1`
- `mipCount`: `0`
- Dimensions: `width = SDFResolution.x`, `height = SDFResolution.y`, `volumeDepth = SDFResolution.z` [S1, unofficial].

**Parameters:**
- `SDFResolution` (Vector3Int): Dimensions of the 3D SDF volume in voxels. Typically set as `DebugWindow.Instance.SdfResolution * 8` [S1, unofficial].
- `SDFRadius` (float): Radius of the SDF brush/kernel used for mutation and Eikonal propagation. Passed to shaders as `_sdfRadius` [S1, unofficial].
- `SDFMaxDist` (float): Maximum distance value to compute or store in the volume. Passed to shaders as `_sdfMaxDist` [S1, unofficial].
- `BrushSize` (float): Radius of the interactive brush [S1, unofficial].
- `BrushHeight` (float): Height parameter for the brush interaction [S1, unofficial].

**Numbered Steps:**
1. **Initialization:** Create two `RFloat` 3D render textures with the target resolution defined by `SDFResolution`. Dispatch the `SDFGen` compute kernel to populate the initial SDF (e.g., a sphere) into the first buffer. Calculate thread group counts as `_numKernels = SDFResolution / kernelSize`, where `kernelSize` is 8 (from `[numthreads(8,8,8)]`) [S1, unofficial].
2. **Buffer Selection:** Determine the output buffer index using bitwise operation: `outSdfTex = (_activeSdfTex + 1) & 2`. Note: The source uses `& 2` which implies a specific indexing scheme or potential typo for `& 1` given two buffers, but the source explicitly states `& 2` [S1, unofficial]. Update `_activeSdfTex = outSdfTex` after dispatch.
3. **Mutation Dispatch:** Bind the input texture `_sdfVolumeTexture[inSdfTex]` and output texture `_sdfVolumeTexture[outSdfTex]` to the `SDFMutate` kernel. Dispatch the kernel with `_numKernels` [S1, unofficial].
4. **Eikonal Passes (if needed):** If the left mouse button is pressed and mode is subtract (or add), calculate the required number of Eikonal propagation passes to fill the cut radius: `_eikonelPasses = (int)(BrushSize / (SDFRadius * 2.0 / SDFResolution.x)) * 2`. Dispatch the `SDFMutate` kernel with `_doEikonel = true` for this many iterations [S1, unofficial].

**Pseudocode (Compute Shader Kernel `SDFMutate`):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for SDFMutate kernel, based on [S1, unofficial]
# Input: _sdfTexIn (RWTexture3D<float>), _sdfTexOut (RWTexture3D<float>)
# Uniforms: _mouseOrigin, _mouseDir, _brushSize, _brushHeight, _mouseSubtract, _mouseAdd, _doEikonel, _sdfRadius, _sdfTexSizeX

def kernelToUV(id):
    # Converts thread ID to UV coordinates in [0, 1]
    # Assuming id is uint3 and _sdfTexSizeX is width
    return float3(id.x / _sdfTexSizeX, id.y / _sdfTexSizeY, id.z / _sdfTexSizeZ)

def uvToWorld(uv):
    # Converts UV to world space coordinates
    # Implementation details not provided in source, assumed to be a standard transform
    return uv * 2.0 - 1.0 # Placeholder for actual world transform

def sdSphere(worldPos, center, radius):
    return length(worldPos - center) - radius

def raySphereIntersect(origin, direction, center, radius):
    # Returns distance to intersection, or large value if no hit
    oc = origin - center
    b = dot(oc, direction)
    c = dot(oc, oc) - radius * radius
    disc = b * b - c
    if disc < 0:
        return 1e10
    t = -b - sqrt(disc)
    return max(t, 0.0)

def opSmoothUnion(d1, d2, k):
    h = clamp(0.5 + 0.5 * (d2 - d1) / k, 0.0, 1.0)
    return mix(d2, d1, h) - k * h * (1.0 - h)

def opSmoothSubtraction(d1, d2, k):
    # Smooth subtraction is typically smooth union of d1 and -d2
    return opSmoothUnion(d1, -d2, k)

def neighborMin(kernel, delta):
    # Accesses neighbor SDF values and returns min absolute value
    # sdfUV is assumed to read from _sdfTexIn
    a = abs(sdfUV(kernelToUV(kernel + delta)))
    b = abs(sdfUV(kernelToUV(kernel - delta)))
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
    xyz = u.x + u.y + u.z
    n = 3.0
    d = (xyz * xyz) - n * ((u.x * u.x + u.y * u.y + u.z * u.z) - (g * g))
    if d < 0.0:
        return False, 0.0
    e = (1.0 / n) * (xyz + sqrt(d))
    return True, e

def eikonal(u, g):
    valid, e = eikonal3d(u, g)
    if valid:
        return e
    e1 = eikonal2d(u.x, u.y, g)
    e2 = eikonal2d(u.x, u.z, g)
    e3 = eikonal2d(u.y, u.z, g)
    return min(min(e1, e2), e3)

def pack(value):
    # Placeholder for packing logic if not direct float assignment
    return value

[numthreads(8, 8, 8)]
def SDFMutate(id: uint3):
    uv = kernelToUV(id)
    world = uvToWorld(uv)
    
    # Initial SDF value (e.g., sphere)
    current = sdSphere(world, float3(0.0, 0.0, 0.0), _sphereRadius)
    
    # Mouse Interaction
    if _mouseAdd or _mouseSubtract:
        toSdf = raySphereIntersect(_mouseOrigin.xyz, _mouseDir.xyz, float3(0.0, 0.0, 0.0), _sphereRadius + _brushHeight)
        ro = _mouseOrigin.xyz + _mouseDir.xyz * toSdf
        d = sdSphere(world, ro, _brushSize * (1.25 if _mouseSubtract else 1.0))
        
        if _mouseAdd:
            current = opSmoothUnion(d, current, 0.025)
        elif _mouseSubtract:
            current = opSmoothSubtraction(d, current, 0.025)
            
    # Eikonal Propagation
    if _doEikonel:
        u = float3(
            neighborMin(id, uint3(1, 0, 0)),
            neighborMin(id, uint3(0, 1, 0)),
            neighborMin(id, uint3(0, 0, 1))
        )
        g = _sdfRadius * 2.0 / _sdfTexSizeX
        e = eikonal(u, g)
        if current > g:
            current = e
            
    _sdfTexOut[id.xyz] = pack(current)
```

**Edge Cases and Pitfalls:**
- **Precision:** The use of `RFloat` (32-bit float) may lead to precision issues for very large volumes or fine details. [G]
- **Boundary Conditions:** The `neighborMin` function accesses neighbors. At volume boundaries, these accesses may read uninitialized or zero values, potentially causing artifacts. [G]
- **Bitwise Operation:** The source uses `outSdfTex = (_activeSdfTex + 1) & 2`. For a two-buffer system, this should likely be `& 1`. If `& 2` is intentional, it implies a different indexing scheme or a bug. [S1, unofficial]
- **Eikonal Passes Calculation:** The formula `_eikonelPasses = (int)(BrushSize / (SDFRadius * 2.0 / SDFResolution.x)) * 2` assumes a specific relationship between brush size, SDF radius, and resolution. Incorrect parameters may lead to insufficient propagation. [S1, unofficial]

## Subtopics

### [SDF Mutation & Eikonal Propagation](4752-sdf-mutation-eikonal-propagation.md)
SDF Mutation & Eikonal Propagation is a GPU-based method for interactively modifying Signed Distance Fields (SDFs) using volumetric brushes. It solves the problem of maintaining geometrically correct distance values after local modifications (addition or subtraction) by combining immediate ray-casting-based brush application with iterative Eikonal equation propagation. It differs from static SDF generation by supporting real-time user input and from simple voxel carving by ensuring the resulting field remains a valid distance function through distance propagation [G].

## Operating characteristics
- **Time Complexity:** O(N) per voxel per pass, where N is the number of voxels in the volume. Eikonal propagation requires multiple passes, scaling with the brush radius and resolution.
- **Memory Complexity:** O(N) for the two ping-pong buffers.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Performance degrades with increased resolution and brush size due to more Eikonal passes.
- **Strengths:** Real-time interactive modification, GPU-accelerated, maintains distance field validity.
- **Limitations:** Limited to GPU execution, potential precision issues, boundary artifacts.
- **When to use:** Interactive SDF editing, real-time volume modification.
- **When not to use:** Static SDF generation, CPU-only environments, high-precision requirements.

## Background
This method originates from the Unity-based implementation by samuelbigos, as documented in the GitHub repository `samuelbigos/signed_distance_fields_in_unity` [S1, unofficial]. It is designed for interactive SDF manipulation within the Unity engine, leveraging compute shaders for parallel processing.

## Sources
[S1] samuelbigos/signed_distance_fields_in_unity, https://github.com/samuelbigos/signed_distance_fields_in_unity, fetched 2026-10-27 (unofficial)
