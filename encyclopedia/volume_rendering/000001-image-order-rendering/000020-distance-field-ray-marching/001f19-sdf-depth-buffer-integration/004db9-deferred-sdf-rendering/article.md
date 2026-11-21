# Deferred SDF Rendering

Also called: Deferred Signed Distance Field rendering, SDF G-buffer pre-pass

## Overview
Deferred SDF Rendering is a technique that decouples Signed Distance Field (SDF) geometry generation from lighting calculations by rendering SDFs to a G-buffer during a pre-pass [S1, unofficial]. This approach allows SDF geometry to participate in standard deferred post-processing effects such as screen-space reflections, ambient occlusion, and bloom, while reducing shader cost and register pressure compared to integrated forward rendering [S1, unofficial]. It differs from integrated SDF rendering by sacrificing high-quality, SDF-specific reflections and shadows in exchange for compatibility with the existing deferred lighting pipeline [S1, unofficial].

## How it works
The method renders SDFs using a full-screen triangle pass during the G-buffer pre-pass, utilizing a modified standard shader [S1, unofficial]. The core challenge is generating correct ray directions for ray marching and outputting depth values that correctly sort against existing polygonal geometry in the depth buffer [S1, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   Vertex position (`v.vertex`) and texture coordinates (`v.texcoord`) from the full-screen triangle.
    *   Camera parameters: `_WorldSpaceCameraPos` (eye position), `InverseViewProjectionMatrix` (inverse of $V \times P$), `UNITY_MATRIX_VP` (view-projection matrix).
    *   SDF field definition (implicit in the shader logic, e.g., sphere, box).
*   **Outputs:**
    *   G-buffer data (albedo, normal, roughness, etc., as defined by the modified standard shader).
    *   Depth value (`outDepth`) written to the depth buffer.

### Data Structures
*   **Vertex Input:** Standard vertex structure containing position and UVs.
*   **Vertex Output:** Structure containing clip-space position (`pos`) and texture coordinates (`tex`).
*   **Ray Data:**
    *   `eye`: `float3` world-space camera position.
    *   `worldPos`: `float3` world-space position on the far plane (derived from UVs).
    *   `worldDir`: `float3` normalized ray direction.

### Steps
1.  **Vertex Shader (Pass-through):**
    The vertex shader passes the input vertex position and texture coordinates to the fragment shader without modification [S1, unofficial].
    $$
    o.pos = v.vertex
    $$
    $$
    o.tex = v.texcoord
    $$

2.  **Fragment Shader: Ray Origin and Direction:**
    Calculate the ray origin as the world-space camera position [S1, unofficial].
    $$
    eye = \text{_WorldSpaceCameraPos}
    $$
    Transform the screen-space texture coordinates (ranging from $[0, 1]$) to clip-space coordinates (ranging from $[-1, 1]$), setting $z=1.0$ (far plane) and $w=1.0$ [S1, unofficial]. Note that the $y$-component must be flipped to match the coordinate system convention [S1, unofficial].
    $$
    clipPos = (2.0 \cdot i.tex.xy - 1.0, 1.0, 1.0)
    $$
    $$
    clipPos.y = -clipPos.y
    $$
    Transform the clip-space position to world space using the inverse view-projection matrix, then perspective divide to obtain the world position on the far plane [S1, unofficial].
    $$
    worldPos = \text{InverseViewProjectionMatrix} \times clipPos
    $$
    $$
    worldPos.xyz = \frac{worldPos.xyz}{worldPos.w}
    $$
    Compute the ray direction by normalizing the vector from the eye to the world position [S1, unofficial].
    $$
    worldDir = \text{normalize}(worldPos.xyz - eye)
    $$

3.  **Ray Marching:**
    Perform standard SDF ray marching using `eye` and `worldDir` to find the hit position (`hitPos`) on the SDF surface [S1, unofficial]. (Implementation details of the SDF field and marching loop are assumed to be provided by the specific SDF shader logic).

4.  **Depth Output:**
    To ensure the SDF geometry correctly occludes other geometry and is occluded by it, project the `hitPos` to Normalized Device Coordinates (NDC) using the view-projection matrix and output the depth [S1, unofficial].
    $$
    clipPos = UNITY\_MATRIX\_VP \times (hitPos, 1.0)
    $$
    $$
    outDepth = \frac{clipPos.z}{clipPos.w}
    $$

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader
struct VertexOutputDeferred {
    float4 pos : SV_POSITION;
    float2 tex : TEXCOORD0;
};

VertexOutputDeferred vertSDFDeferred(float4 v.vertex : POSITION, float2 v.texcoord : TEXCOORD0) {
    VertexOutputDeferred o;
    o.pos = v.vertex;
    o.tex = v.texcoord;
    return o;
}

# Fragment Shader
float4 fragSDFDeferred(VertexOutputDeferred i) : SV_TARGET {
    // 1. Calculate Ray Origin
    float3 eye = _WorldSpaceCameraPos;

    // 2. Calculate Ray Direction
    // Map UV [0,1] to Clip Space [-1,1], set z=1 (far plane), flip Y
    float4 clipPos = float4(i.tex.xy * 2.0 - 1.0, 1.0, 1.0);
    clipPos.y = -clipPos.y;

    // Transform to World Space
    float4 worldPos = mul(InverseViewProjectionMatrix, clipPos);
    worldPos.xyz = worldPos.xyz / worldPos.w;

    // Normalize direction
    float3 worldDir = normalize(worldPos.xyz - eye);

    // 3. Ray Marching (Placeholder for specific SDF logic)
    // float hitDist = RayMarchSDF(eye, worldDir);
    // float3 hitPos = eye + worldDir * hitDist;
    
    // For the purpose of depth output, assume hitPos is calculated:
    float3 hitPos = ...; // Result of SDF ray march

    // 4. Output Depth
    // Project hitPos to NDC
    float4 clipHitPos = mul(UNITY_MATRIX_VP, float4(hitPos, 1.0));
    float outDepth = clipHitPos.z / clipHitPos.w;

    // Write depth to SV_Depth (or equivalent depth target)
    // Note: In Unity's deferred pre-pass, depth is often handled automatically 
    // if SV_DEPTH is used, or via a separate depth pass. 
    // The source explicitly calculates outDepth for this purpose.
    
    // Output G-buffer data (Albedo, Normal, etc.)
    // ...
    
    return float4(0, 0, 0, 1); // Placeholder for G-buffer output
}
```

### Parameters and Edge Cases
*   **Far Plane:** The ray direction calculation assumes a far plane depth of $1.0$ in clip space. If the camera's far plane is significantly different, the ray direction accuracy may degrade for distant objects, though normalization mitigates this for direction.
*   **Coordinate System:** The $y$-flip in clip space is critical. Failure to flip $y$ results in inverted ray directions, causing SDFs to render incorrectly or not at all [S1, unofficial].
*   **Depth Precision:** Writing depth for ray-marched surfaces requires careful handling of floating-point precision, especially for thin SDF features or very close/far distances.

### Pitfalls
*   **Lighting Quality:** Decoupling lighting from SDF rendering prevents the production of high-quality reflections and shadows specific to the SDF geometry [S1, unofficial].
*   **Shadow Cost:** Rendering SDFs into shadowmaps is possible but significantly increases shadow cost and may not be supported by all deferred pipelines without custom modifications [S1, unofficial].
*   **Material Control:** Using the standard material system applies settings (roughness, emissive) uniformly to all SDFs rendered with that material, lacking fine-grained control per SDF instance [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The pre-pass adds a full-screen triangle draw call. The fragment shader cost is dominated by the SDF ray marching loops and branches. However, by decoupling lighting, the lighting calculation itself is not repeated per SDF pixel in the main lighting pass, potentially reducing overall register pressure and shader complexity [S1, unofficial].
*   **Memory Complexity:** Requires standard G-buffer storage. No additional memory for SDF-specific lighting data is needed in the deferred pipeline.
*   **Strengths:**
    *   Integrates SDFs into existing deferred post-processing effects (SSR, SSAO, Bloom) [S1, unofficial].
    *   Reduces shader register pressure by separating geometry generation from lighting [S1, unofficial].
    *   Correctly sorts SDFs against polygonal geometry via depth buffer integration [S1, unofficial].
*   **Limitations:**
    *   Lacks high-quality, SDF-specific reflections and shadows [S1, unofficial].
    *   Shadow map rendering for SDFs is expensive and non-trivial [S1, unofficial].
    *   Limited material control granularity when using standard shaders [S1, unofficial].
*   **When to Use:** When SDFs are used for environmental geometry or effects that benefit from screen-space post-processing, and high-fidelity SDF-specific lighting (like accurate SDF shadows) is not required.
*   **When Not to Use:** When high-quality reflections or shadows on SDF geometry are critical, or when fine-grained material control per SDF instance is needed.

## Background
This technique was demonstrated by the author of "Interplay of Light" in December 2017, using Unity 2017.2 [S1, unofficial]. It was developed as an experiment to integrate SDFs into an actual rendering engine context, leveraging Unity's material system and deferred rendering pipeline. The SDF implementations referenced were based on external posts (not detailed in the source), and the approach was described as a "fun exercise" and "hacky" but functional for integrating SDFs with post-processing effects [S1, unofficial].

## Sources
[S1] Deferred Signed Distance Field rendering – Interplay of Light, https://interplayoflight.wordpress.com/2017/12/12/deferred-signed-distance-field-rendering/, fetched 2026-10-30 (unofficial)
