# Debug Splat Point Rendering

Also called: Debug Point Rendering, Gaussian Splatting/Debug/Render Points, GaussianDebugRenderPoints, Splat Center Visualization

## Overview
Debug Splat Point Rendering is a diagnostic visualization technique that renders individual 3D Gaussian splats as small screen-space quads centered at their projected positions [S1, S2, unofficial]. Unlike the production rendering path which draws full elliptical splats for opacity blending, this method isolates the center location and basic attributes of each splat, allowing developers to inspect splat distribution, density, and attribute correctness [S2, unofficial]. It serves as a lightweight alternative to full splat rendering for verifying the correctness of the splat data pipeline and camera projection logic [S1, S2, unofficial].

## How it works
The method renders one quad per splat using a single draw call with instanced rendering [S2, unofficial]. It bypasses the complex spherical harmonic color evaluation and alpha blending of the main renderer, opting instead for either the raw SH color or a normalized index-based color map [S1, S2, unofficial].

### Inputs and Outputs
**Inputs:**
*   **Splat Data Buffer:** A structured buffer containing splat attributes [S2, unofficial]. Each splat entry includes `centerWorldPos` (world-space position) and `sh.col` (spherical harmonic color coefficient) [S2, unofficial].
*   **Shader Parameters:** `_SplatSize` (float, quad size in screen pixels), `_DisplayIndex` (bool, toggle color mode), and `_SplatCount` (int, total splats for normalization) [S2, unofficial].

**Outputs:**
*   **Framebuffer:** A 2D image where each splat appears as a small square at its projected center [S2, unofficial].

### Data Structures
The vertex shader processes instances using `uint vtxID` (SV_VertexID) and `uint instID` (SV_InstanceID) [S2, unofficial]. The fragment shader receives a `v2f` struct containing `half3 color` (TEXCOORD0) and `float4 vertex` (SV_POSITION) [S2, unofficial].

### Numbered Steps

1.  **Retrieve Splat Data:**
    For each instance, load splat attributes using the instance ID as the buffer index: `splat = LoadSplatData(instID)` [S2, unofficial].

2.  **Transform to Clip Space:**
    Transform the splat's world position to clip space to determine the quad center: `centerWorld = splat.centerWorldPos`, `centerClip = UNITY_MATRIX_VP * float4(centerWorld, 1.0)` [S1, S2, unofficial].

3.  **Generate Quad Geometry:**
    Generate a 2x2 quad in screen space centered on the projected vertex [S2, unofficial]. Calculate a 2D offset vector `quadPos` based on `vtxID`: `quadPos.x = (vtxID & 1) * 2.0 - 1.0`, `quadPos.y = ((vtxID >> 1) & 1) * 2.0 - 1.0` [S2, unofficial].

4.  **Apply Screen-Space Offset:**
    Offset the vertex position in screen space by scaling `quadPos` with `_SplatSize`, normalizing by `_ScreenParams.xy`, and multiplying by the clip-space W component: `offset = quadPos * (_SplatSize / _ScreenParams.xy) * centerClip.w` [S2, unofficial]. Update vertex XY: `outputVertex.xy = centerClip.xy + offset`, preserving Z and W [S1, S2, unofficial].

5.  **Compute Fragment Color:**
    Determine the color based on `_DisplayIndex` [S2, unofficial]. If false, use raw SH color: `color = saturate(splat.sh_col)` [S2, unofficial]. If true, encode the index into RGB: `normalizedIndex = instID / _SplatCount`, `r = frac(normalizedIndex * 100)`, `g = frac(normalizedIndex * 10)`, `b = normalizedIndex` [S2, unofficial].

6.  **Output:**
    Pass the computed color and clip-space position to the fragment shader [G]. The fragment shader outputs the interpolated color [G].

### Pseudocode (Python-like)

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vertex_shader(vtx_id: int, inst_id: int) -> dict:
    # [G] Loads splat data from buffer using instance ID
    splat = load_splat_data(inst_id)

    # [S1, S2, unofficial] Transform to clip space
    center_world = splat.center_world_pos
    center_clip = mat_mul(UNITY_MATRIX_VP, float4(center_world, 1.0))

    # [S2, unofficial] Generate 2x2 quad offset based on vertex ID
    quad_pos_x = (vtx_id & 1) * 2.0 - 1.0
    quad_pos_y = ((vtx_id >> 1) & 1) * 2.0 - 1.0
    quad_pos = float2(quad_pos_x, quad_pos_y)

    # [S1, S2, unofficial] Apply screen-space offset
    offset = quad_pos * (_SplatSize / _ScreenParams.xy) * center_clip.w
    output_vertex = float4(center_clip.xy + offset, center_clip.z, center_clip.w)

    # [S2, unofficial] Compute color based on display mode
    if _DisplayIndex:
        normalized_index = inst_id / _SplatCount
        color_r = frac(normalized_index * 100.0)
        color_g = frac(normalized_index * 10.0)
        color_b = normalized_index
        color = float3(color_r, color_g, color_b)
    else:
        color = saturate(splat.sh_col)

    return {"vertex": output_vertex, "color": color}

def fragment_shader(input_data: dict) -> float4:
    # [G] Outputs interpolated color with full opacity
    return float4(input_data["color"], 1.0)
```

### Parameters and Typical Values
*   `_SplatSize`: Float scalar defining the width/height of the debug quad in screen pixels [S2, unofficial].
*   `_DisplayIndex`: Boolean flag. If true, colors represent splat indices; if false, colors represent SH color [S2, unofficial].
*   `_SplatCount`: Integer scalar, the total number of splats, used for normalizing indices [S2, unofficial].

### Edge Cases and Pitfalls
*   **Quad Generation Assumption:** The method assumes a fixed 2x2 grid per splat (4 vertices) [S2, unofficial].
*   **Division by Zero:** If `_SplatCount` is zero, division by zero occurs during index encoding [S2, unofficial].
*   **Precision:** Using `frac()` for index encoding can lead to visual banding if `_SplatCount` is very large, as floating-point precision limits distinct adjacent indices [S1, S2, unofficial].

## Subtopics

### Debug Point Cloud Rendering
A shader for rendering debug point clouds with configurable size, color, and alpha, using spatial rendering mode [S3, unofficial]. Parameters include `point_size` (float, default 10.0), `color` (vec3, default cyan `vec3(0.0, 1.0, 1.0)`), and `alpha` (float, default 0.1) [S3, unofficial]. The vertex shader sets `POINT_SIZE = point_size`, and the fragment shader outputs `ALBEDO = color` with `ALPHA = alpha` [S3, unofficial].

## Operating characteristics
*   **Time and memory complexity:** Not given in the sources.
*   **Measured performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths and limitations:** Not given in the sources.
*   **When to use it and when not:** Not given in the sources.

## Background
Debug Splat Point Rendering is a specialized technique developed within real-time 3D Gaussian Splatting implementations for Unity, utilizing `unity_ObjectToWorld` and `UNITY_MATRIX_VP` matrices [S1, S2, unofficial]. It emerged as a diagnostic tool for developers to inspect splat positions and attributes during pipeline development [S2, unofficial]. The method is used for verifying that splat data has been correctly loaded, transformed, and projected [S2, unofficial].

## Sources
[S1] aras-p/UnityGaussianSplatting, https://github.com/aras-p/UnityGaussianSplatting, fetched 2026-10-26 (unofficial)
[S2] nigelhartman/worldlabs_unity, https://github.com/nigelhartman/worldlabs_unity, fetched 2026-10-27 (unofficial)
[S3] dsvalerian/better-procedural-terrain, https://github.com/dsvalerian/better-procedural-terrain, fetched 2026-10-27 (unofficial)
