# Quarter-Resolution Temporal Upsampling

Also called: Quarter-Resolution Temporal Upsampling for Volumetric Clouds, Temporal Reprojection Anti-Aliasing for Clouds, Upsampling to Improve Volumetric Cloud Render Performance

## Overview
Quarter-Resolution Temporal Upsampling is an optimization technique for real-time volumetric cloud rendering that reduces computational load by rendering to a buffer at one-quarter resolution (1/16th of total pixels) and upscaling to full resolution using temporal reprojection and pixel jittering to maintain visual fidelity [S1, unofficial]. Unlike selective fragment rendering which stalls GPU waves by drawing sparse pixels, this method renders a full quarter-resolution image, utilizing all threads efficiently while avoiding wave stalls [S1, unofficial]. It sets itself apart from standard bilinear upsampling by using jitter and temporal blending to fill missing gaps, preserving detail over 16 frames rather than simply stretching the low-res image [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Full-resolution render target dimensions (e.g., 2048x2048), previous frame's upsampled color and depth buffers, current frame's quarter-resolution color and depth buffers, and a pre-computed 16-frame jitter sequence.
*   **Output:** Full-resolution upsampled and blended color and depth buffers.

**Data Structures:**
1.  **Quarter Resolution Buffers:** Two render targets (Color and Depth) with dimensions $W/4 \times H/4$.
2.  **Double Buffer:** Two full-resolution render targets (Color and Depth) acting as Front (previous frame result) and Back (current frame result).
3.  **Jitter Array:** A static array of 16 `Vector2` offsets, each containing integer coordinates $(x, y)$ where $x, y \in [0, 3]$.

**Numbered Steps:**
1.  **Render Quarter-Resolution Clouds:**
    *   Clear the quarter-resolution color and depth buffers.
    *   Execute the volumetric cloud raymarching fragment shader.
    *   In the shader, calculate the ray direction using the current UV coordinate plus a frame-specific jitter offset.
    *   Write the resulting color and depth to the quarter-resolution buffers.
2.  **Upsample and Mix:**
    *   Set the upsample shader parameters: reference to the quarter-resolution current buffer, the full-resolution previous buffer, the current jitter offset, and the full resolution width/height.
    *   Perform a full-screen blit from the previous full-resolution buffer to the current full-resolution back buffer using the upsample shader.
    *   The shader calculates a jitter correction weight for each pixel to determine how much to blend between the current quarter-res sample (upsampled) and the previous full-res sample.
    *   Apply a convergence speed factor based on the pixel's distance from the zenith to blend the jitter-corrected color with the previous frame's color.
3.  **Swap Buffers:**
    *   Swap the Front and Back double buffers for the next frame.
    *   **Crucial:** Do **not** clear the new Back buffer before the swap. The texture may be sampled by other passes (shadows, reflections, hemisphere mesh) within the same frame. Clearing it would cause those passes to read cleared data instead of the current render due to timing issues [S1, unofficial].

**Formulas:**

*   **Ray Direction Calculation with Jitter:**
    The jitter is applied to the UV coordinates before converting to spherical coordinates for ray direction.
    $$ uv_{jitter} = \frac{FrameJitter}{Resolution} $$
    $$ longitude = (uv + uv_{jitter}) - 0.5 $$
    $$ l = \text{length}(longitude) $$
    $$ a = l \times \pi $$
    $$ sin1 = \sin(a) $$
    $$ sin2 = \frac{longitude.y}{l} $$
    $$ cos1 = \cos(a) $$
    $$ cos2 = \frac{longitude.x}{l} $$
    $$ SphericalRayDirectionFromUV(uv) = \text{normalize}(\text{float3}(sin1 \times cos2, cos1, sin1 \times sin2)) $$
    Where $FrameJitter$ is the current frame's 2D offset vector, and $Resolution$ is the full-size render target dimension (e.g., 2048), not the quarter buffer dimension [S1, unofficial].

*   **Jitter Correction Weight:**
    Determines if the current full-resolution pixel "matches" the low-resolution one rendered this frame.
    $$ localIndex = \text{floor}(\text{fmod}(uv \times Resolution, 4.0)) $$
    $$ localIndex = |localIndex - FrameJitter| $$
    $$ w = \text{saturate}(localIndex.x + localIndex.y) $$
    Where $w=0$ indicates the pixel matches the current quarter-res render (take from current), and $w=1$ indicates it should take from the previous frame [S1, unofficial].

*   **Final Color Blend:**
    $$ C_{jittered} = \text{lerp}(C_{curr}, C_{prev}, w) $$
    $$ normalizedUV = (uv \times 2.0) - 1.0 $$
    $$ distanceFromZenith = \text{saturate}(\text{length}(normalizedUV)) $$
    $$ C_{speed} = \text{lerp}(S_{zenith}, S_{horizon}, distanceFromZenith) $$
    $$ C_{final} = \text{lerp}(C_{prev}, C_{jittered}, C_{speed}) $$
    Where $S_{zenith} = 0.75$ and $S_{horizon} = 0.5$ are arbitrary convergence speeds [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pre-computed jitter array (16 frames)
FrameJitters = [
    (0, 2), (0, 1), (3, 1), (1, 2),
    (0, 3), (1, 0), (1, 3), (1, 1),
    (2, 0), (2, 1), (3, 2), (0, 0),
    (2, 3), (3, 0), (2, 2), (3, 3)
]

def RenderQuarterSizeClouds(CommandBuffer, QuarterColorRT, QuarterDepthRT, CloudMaterial):
    # Clear only the quarter buffers
    QuarterColorRT.Clear()
    QuarterDepthRT.Clear()
    
    # Rasterize quad with CloudMaterial
    # CloudMaterial uses FrameJitters[FrameCount % 16] for jitter
    CommandBuffer.DrawQuad(QuarterColorRT, QuarterDepthRT, CloudMaterial)

def UpsampleAndMixClouds(CommandBuffer, PrevFullColorRT, PrevFullDepthRT, CurrQuarterColorRT, CurrQuarterDepthRT, NextFullColorRT, NextFullDepthRT, FrameJitter, Resolution):
    # Set upsample material parameters
    UpsampleMaterial.SetTexture("_CurrFrameColor", CurrQuarterColorRT)
    UpsampleMaterial.SetTexture("_PrevFrameColor", PrevFullColorRT)
    UpsampleMaterial.SetTexture("_PrevFrameDepth", PrevFullDepthRT)
    UpsampleMaterial.SetVector("_FrameJitter", FrameJitter)
    UpsampleMaterial.SetFloat("_Resolution", Resolution)
    
    # Blit from Prev to Next using UpsampleMaterial
    # Note: Do NOT clear NextFullColorRT or NextFullDepthRT before this
    CommandBuffer.Blit(PrevFullColorRT, NextFullColorRT, UpsampleMaterial)
    CommandBuffer.Blit(PrevFullDepthRT, NextFullDepthRT, UpsampleMaterial)

def CloudRaymarchShader(uv):
    # Apply jitter to UV
    uv_jitter = FrameJitter / Resolution
    longitude = (uv + uv_jitter) - 0.5
    l = length(longitude)
    a = l * PI
    sin1 = sin(a)
    sin2 = longitude.y / l
    cos1 = cos(a)
    cos2 = longitude.x / l
    
    ray_dir = normalize(float3(sin1 * cos2, cos1, sin1 * sin2))
    
    # Perform raymarching and return color/depth
    return Raymarch(ray_dir)

def UpsampleShader(uv):
    # Sample current quarter-res (upsampled via bilinear)
    curr_color = _CurrFrameColor.Sample(uv)
    curr_depth = _CurrFrameDepth.Sample(uv)
    
    # Sample previous full-res
    prev_color = _PrevFrameColor.Sample(uv)
    prev_depth = _PrevFrameDepth.Sample(uv)
    
    # Calculate jitter correction
    local_index = floor(fmod(uv * _Resolution, 4.0))
    local_index = abs(local_index - _FrameJitter)
    w = saturate(local_index.x + local_index.y)
    
    # Blend current and previous based on jitter match
    jittered_color = lerp(curr_color, prev_color, w)
    jittered_depth = lerp(curr_depth, prev_depth, w)
    
    # Calculate convergence speed based on distance from zenith
    normalized_uv = (uv * 2.0) - 1.0
    distance_from_zenith = saturate(length(normalized_uv))
    convergence_speed = lerp(0.75, 0.5, distance_from_zenith)
    
    # Final blend
    final_color = lerp(prev_color, jittered_color, convergence_speed)
    final_depth = lerp(prev_depth, jittered_depth, convergence_speed)
    
    return final_color, final_depth

def MainLoop():
    frame_count = 0
    PrevColorRT, PrevDepthRT = CreateDoubleBuffer()
    NextColorRT, NextDepthRT = CreateDoubleBuffer()
    QuarterColorRT, QuarterDepthRT = CreateQuarterBuffers()
    
    while True:
        jitter = FrameJitters[frame_count % 16]
        
        # 1. Render quarter resolution
        RenderQuarterSizeClouds(CommandBuffer, QuarterColorRT, QuarterDepthRT, CloudMaterial)
        
        # 2. Upsample and mix
        UpsampleAndMixClouds(CommandBuffer, PrevColorRT, PrevDepthRT, QuarterColorRT, QuarterDepthRT, NextColorRT, NextDepthRT, jitter, Resolution)
        
        # 3. Swap buffers (Do not clear Next buffers)
        Swap(PrevColorRT, NextColorRT)
        Swap(PrevDepthRT, NextDepthRT)
        
        frame_count += 1
```

**Parameters:**
*   **FrameJitter:** A pre-computed array of 16 `Vector2` offsets that visit each pixel in a 4x4 block. Example values: `(0,2), (0,1), (3,1), (1,2), (0,3), (1,0), (1,3), (1,1), (2,0), (2,1), (3,2), (0,0), (2,3), (3,0), (2,2), (3,3)` [S1, unofficial].
*   **Resolution:** The full-size render target resolution (e.g., 2048), used to normalize the jitter offset. It is NOT the quarter buffer resolution [S1, unofficial].
*   **Convergence Speeds:** $S_{zenith}=0.75$, $S_{horizon}=0.5$. These are arbitrary values that control how quickly the temporal blend converges to the new data [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Buffer Clearing:** Do not clear the back double buffer before swapping. Clearing it can cause other rendering passes (hemisphere mesh, shadow maps, reflection probes) that sample the texture within the same frame to read cleared data instead of the current render due to timing issues [S1, unofficial].
*   **Wave Stalls:** Avoid selective fragment rendering (drawing 1 pixel per 4x4 group per frame) as it stalls GPU waves/warps because threads execute in batches (16, 32, 64). This method renders a full quarter-resolution image, utilizing all threads efficiently [S1, unofficial].
*   **Jitter Resolution:** The `_Resolution` parameter in the jitter calculation must be the full-size target resolution, not the quarter buffer resolution, to correctly counteract the movement in the upsample [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Rendering time is proportional to $1/16$ of the full resolution, plus the cost of a full-resolution upsample pass. The upsample pass is significantly cheaper than full volumetric raymarching.
*   **Memory Complexity:** Requires three render targets: one quarter-resolution color/depth pair, and one full-resolution double buffer (color/depth).
*   **Measured Performance:** For a 2048x2048 target, rendering to a 512x512 buffer reduces pixel count from 4,194,304 to 262,144 [S1, unofficial].
*   **Scaling:** Performance scales linearly with the reduction in resolution. The visual quality scales with the number of frames accumulated (16 frames for full detail).
*   **Strengths:** Efficient GPU utilization (no wave stalls), significant performance gain, maintains visual fidelity over time through temporal accumulation.
*   **Limitations:** Requires 16 frames to fully converge and eliminate artifacts, which can cause ghosting or smearing in fast-moving scenes or during rapid camera movements. Requires careful handling of buffer swapping to avoid sampling cleared data.
*   **When to use:** Real-time volumetric cloud rendering where performance is critical and motion is moderate.
*   **When not to use:** Scenes with extremely fast camera movement or rapid changes in cloud appearance, where temporal ghosting would be noticeable.

## Background
This technique was popularized in the context of real-time volumetric cloud rendering, specifically referenced in "Optimizations for Real-Time Volumetric Cloudscapes" and discussed in detail by Vertex Fragment in "Upsampling to Improve Volumetric Cloud Render Performance" [S1, unofficial]. It addresses the high cost of raymarching clouds by reducing the number of rays cast per frame while maintaining image quality through temporal reprojection and jittering. It is an alternative to selective fragment rendering, which suffers from GPU warp stalls [S1, unofficial].

## Sources
[S1] Upsampling to Improve Volumetric Cloud Render Performance · Vertex Fragment, https://www.vertexfragment.com/ramblings/volumetric-cloud-upsampling/, fetched 2026-10-30 (unofficial)
