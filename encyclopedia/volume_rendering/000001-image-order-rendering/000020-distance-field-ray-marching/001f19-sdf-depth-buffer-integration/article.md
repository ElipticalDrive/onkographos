# SDF Depth Buffer Integration

Also called: Raymarching with Depth Buffer Output, Sphere Tracing for Depth Output, Sphere Tracing with Depth Buffer Output

## Overview
SDF Depth Buffer Integration is a technique that extends sphere tracing by explicitly writing the computed surface hit point's depth to the hardware depth buffer (`SV_Depth`) [S1, unofficial][S2, unofficial][S3, unofficial]. It solves the problem of incorrect occlusion between ray-marched SDF geometry and standard rasterized geometry, where the depth buffer would otherwise record the depth of the proxy mesh face rather than the actual ray-marched surface [S2, unofficial]. It is distinguished from standard raymarching, which typically only outputs color, by ensuring the SDF geometry participates in standard depth testing [S3, unofficial].

## How it works
The method builds on [Distance-field ray marching](21-distance-field-ray-marching.md). It requires a fragment shader that outputs both color and depth, and a specific render queue configuration to ensure correct occlusion ordering.

### Inputs and Outputs
**Inputs:**
- Vertex position in object space.
- Camera position in world space.
- Scene SDF function $S(p)$.

**Outputs:**
- `color` (SV_TARGET): The rendered surface color.
- `depth` (SV_Depth): The NDC depth value of the hit point.

### Data Structures
The vertex shader output struct (`v2f`) layout is:
- `position` (SV_POSITION): Clip-space vertex position for rasterization.
- `localPosition` (TEXCOORD0): Object-space position, used as the ray origin.
- `viewDirection` (TEXCOORD1): Object-space view vector from camera to vertex, stored un-normalized.

### Parameters
- `MAX_STEPS`: Maximum ray march iterations. Typical value: 32 [S2, unofficial][S3, unofficial].
- `THICKNESS`: SDF value threshold for hit detection. Typical value: 0.001 [S2, unofficial][S3, unofficial].
- `NORMAL_EPSILON`: Offset for finite difference normal estimation. Typical value: 0.01 [S2, unofficial][S3, unofficial].

### Steps
1. **Vertex Shader Setup**:
   - Compute clip-space position: `o.position = UnityObjectToClipPos(v.vertex)`.
   - Store object-space vertex position as ray origin: `o.localPosition = v.vertex`.
   - Compute camera position in object space: `objectSpaceCameraPos = mul(unity_WorldToObject, float4(_WorldSpaceCameraPos, 1))`.
   - Compute view direction: `o.viewDirection = v.vertex - objectSpaceCameraPos`.

2. **Fragment Shader Initialization**:
   - Set ray origin `pos = i.localPosition`.
   - Normalize view direction: `dir = normalize(i.viewDirection.xyz)`.
   - Initialize `progress = 0`.
   - Initialize `hitsurface = false`.

3. **Ray Marching Loop**:
   - Iterate `iter` from 0 to `MAX_STEPS - 1`.
   - Calculate sample point: `samplePoint = pos + dir * progress`.
   - Query SDF: `distance = scene(samplePoint)`.
   - If `distance < THICKNESS`:
     - Set `hitsurface = true`.
     - Break loop.
   - Advance ray: `progress = progress + distance`.

4. **Hit/Miss Handling**:
   - If `!hitsurface`: Execute `discard` to prevent writing to buffers [S2, unofficial][S3, unofficial].

5. **Color Calculation**:
   - Compute surface normal using central finite differences:
     $$n_x = S(p + \epsilon e_x) - S(p - \epsilon e_x)$$
     $$n_y = S(p + \epsilon e_y) - S(p - \epsilon e_y)$$
     $$n_z = S(p + \epsilon e_z) - S(p - \epsilon e_z)$$
     where $\epsilon = \text{NORMAL\_EPSILON}$ [S3, unofficial].
   - Transform normal to world space and normalize.
   - Compute lighting and return `color` [S3, unofficial].

6. **Depth Calculation**:
   - Transform hit point to clip space: `tracedClipPos = UnityObjectToClipPos(float4(samplePoint, 1.0))`.
   - Compute NDC depth: `depth = tracedClipPos.z / tracedClipPos.w` [S2, unofficial][S3, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader
def vert(v_vertex):
    o_position = UnityObjectToClipPos(v_vertex)
    o_localPosition = v_vertex
    objectSpaceCameraPos = mul(unity_WorldToObject, float4(_WorldSpaceCameraPos, 1))
    o_viewDirection = v_vertex - objectSpaceCameraPos
    return {
        'position': o_position,
        'localPosition': o_localPosition,
        'viewDirection': o_viewDirection
    }

# Fragment Shader
def frag(i):
    pos = i['localPosition']
    dir = normalize(i['viewDirection'].xyz)
    progress = 0.0
    samplePoint = float3(0, 0, 0)
    hitsurface = False
    
    for iter in range(MAX_STEPS):
        samplePoint = pos + dir * progress
        distance = scene(samplePoint)
        
        if distance < THICKNESS:
            hitsurface = True
            break
        
        progress = progress + distance
    
    if not hitsurface:
        discard()
    
    # Color Calculation
    normal = compute_normal(samplePoint)
    color = render_surface(samplePoint, normal)
    
    # Depth Calculation
    tracedClipPos = UnityObjectToClipPos(float4(samplePoint, 1.0))
    depth = tracedClipPos.z / tracedClipPos.w
    
    return color, depth

def scene(pos):
    # Example: Sphere of radius 0.5 at origin
    return length(pos) - 0.5

def compute_normal(pos):
    eps = NORMAL_EPSILON
    changeX = scene(pos + float3(eps, 0, 0)) - scene(pos - float3(eps, 0, 0))
    changeY = scene(pos + float3(0, eps, 0)) - scene(pos - float3(0, eps, 0))
    changeZ = scene(pos + float3(0, 0, eps)) - scene(pos - float3(0, 0, eps))
    surfaceNormal = float3(changeX, changeY, changeZ)
    surfaceNormal = mul(unity_ObjectToWorld, float4(surfaceNormal, 0))
    return normalize(surfaceNormal)

def render_surface(pos, normal):
    # Simplified lighting
    lightDirection = normalize(_WorldSpaceLightPos0.xyz)
    lightAngle = max(dot(normal, lightDirection), 0.0)
    return _Color * lightAngle * _LightColor0
```

### Edge Cases and Pitfalls
- **Incorrect Occlusion**: Without `SV_Depth` output, the depth buffer records the proxy mesh face depth, causing rasterized objects behind the SDF surface to appear in front, and vice versa [S2, unofficial].
- **Early-Z Disable**: Writing `SV_Depth` disables GPU early-Z optimization for affected pixels because the hardware cannot know the depth before the fragment shader runs [S2, unofficial].
- **Discard vs Clip**: Use `discard` instead of `clip(-1)` to avoid potential driver differences in depth buffering [S2, unofficial][S3, unofficial].
- **Precision Loss**: View direction is stored un-normalized in `TEXCOORD1` and normalized in the fragment shader to avoid precision loss during interpolation [S3, unofficial].
- **Batching**: `DisableBatching` must be set to `True` because batching breaks the object-to-world transform used for ray setup [S2, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N \cdot M)$ where $N$ is the number of pixels and $M$ is `MAX_STEPS`. Each pixel requires up to `MAX_STEPS` SDF evaluations.
- **Memory Complexity**: $O(1)$ per pixel for the ray marching state.
- **Performance**: The explicit depth write disables early-Z, potentially reducing performance compared to standard rasterization or raymarching without depth output [S2, unofficial].
- **Strengths**: Correct occlusion between SDF and rasterized geometry.
- **Limitations**: Loss of early-Z optimization; requires specific render queue settings (`Geometry+1`).

## Background
The method was introduced in Tutorial 045 of the Ronja Tutorials series, extending previous sphere tracing tutorials (043, 044) to address depth buffer integration [S1, unofficial][S2, unofficial][S3, unofficial]. It is commonly used in Unity shader development for rendering SDF geometry alongside standard meshes [S2, unofficial][S3, unofficial].

## Sources
[S1] Sphere Tracing and Ray Marching, https://deepwiki.com/ronja-tutorials/ShaderTutorials/5-sphere-tracing-and-ray-marching, fetched 2026-10-27 (unofficial)
[S2] Sphere Tracing with Depth Buffer Output, https://deepwiki.com/ronja-tutorials/ShaderTutorials/5.3-sphere-tracing-with-depth-buffer-output, fetched 2026-10-27 (unofficial)
[S3] Assets/045_SphereTracingDepth/SphereTracingDepth.shader, https://github.com/ronja-tutorials/ShaderTutorials/blob/31899f7b/Assets/045_SphereTracingDepth/SphereTracingDepth.shader, fetched 2026-10-27 (unofficial)
