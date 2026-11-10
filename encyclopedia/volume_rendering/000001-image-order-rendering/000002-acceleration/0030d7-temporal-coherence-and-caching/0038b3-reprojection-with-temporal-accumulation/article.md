# Reprojection with Temporal Accumulation

Also called: Temporal Anti-Aliasing (TAA) via reprojection

## Overview
Reprojection with Temporal Accumulation stabilizes voxel rendering over time by blending the current frame's color with an accumulated color buffer from the previous frame, using camera motion vectors to align pixels [S1, unofficial]. It differs from sibling methods like [Inter-Ray Coherency](1030-inter-ray-coherency.md) by operating in image space to reduce noise and flickering across frames rather than accelerating ray traversal, and from [Image-Based Rendering for Volume Surfaces](1383-image-based-rendering-for-volume-surfaces.md) by relying on explicit motion vectors for alignment rather than view-dependent interpolation of surface features.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Current frame color buffer (`colour_attachment`), current frame world position buffer (`position_attachment`), previous frame accumulation buffer (`accumulation_attachment`), camera matrices for current and previous frames (`trace_uniforms.current_camera`, `trace_uniforms.last_camera`), and parameters `reprojection_factor` and `indirect_lighting`.
*   **Outputs:** Final output color buffer.

**Data Structures:**
*   `colour_attachment`: `texture_2d<f32>`, binding 3. Stores current frame RGB color.
*   `position_attachment`: `texture_storage_2d<rgba32float, read_write>`, binding 3. Stores world-space positions for each pixel.
*   `accumulation_attachment`: `texture_storage_2d<rgba16float, read_write>`, binding 1. Stores the accumulated color from the previous frame.

**Steps:**
1.  **Load Current Data:** Read the current frame's color $C_{curr}$ from `colour_attachment` at sample position $(u, v)$. Read the current frame's world position $P_{world}$ from `position_attachment` at $(u, v)$.
2.  **Guard Check:** If `indirect_lighting` is zero OR `reprojection_factor` $\le 0.0$, skip reprojection and return $C_{curr}$ directly [S1, unofficial].
3.  **Compute Local Bounds:** To detect edges and prevent bleeding, compute local color bounds using the current pixel and its 8 neighbors ($sample_0$ to $sample_8$).
    *   Calculate `min_col` as the average of the minimum values in the 'plus' pattern (cross: center, up, down, left, right).
    *   Calculate `max_col` as the average of the maximum values in the 'square' pattern (3x3 block).
    *   Let $B_{min}$ and $B_{max}$ be these computed bounds [S1, unofficial].
4.  **Reproject Position:** Transform $P_{world}$ to the previous frame's clip space using `trace_uniforms.last_camera`.
    *   $P_{clip} = M_{last} \times P_{world}$
    *   Convert $P_{clip}$ to Normalized Device Coordinates (NDC): $NDC = P_{clip} / P_{clip.w}$.
    *   Convert NDC to texture coordinates for the accumulation buffer: $T_{proj} = (NDC.xy + 1.0) / 2.0$.
5.  **Validate Projection:** Check if $T_{proj}$ is within the valid NDC range $[-1, 1]$ (or texture coordinates $[0, 1]$). If the projected position is outside this range (indicating the pixel is newly visible or occluded), treat the history as invalid. Set $C_{hist} = C_{curr}$ and skip blending [S1, unofficial].
6.  **Fetch History:** Load the accumulated color $C_{hist}$ from `accumulation_attachment` at $T_{proj}$.
7.  **Blend:** Compute the mixed color $C_{mixed}$ using the `reprojection_factor` (denoted as $\alpha$ or `mix_amount` in sources):
    $$ C_{mixed} = C_{curr} \times (1.0 - \alpha) + C_{hist} \times \alpha $$
    where $\alpha = \text{reprojection\_factor}$ [S1, unofficial].
8.  **Clamp:** Clamp $C_{mixed}$ to the local bounds $[B_{min}, B_{max}]$ to prevent ghosting and color bleeding across edges:
    $$ C_{final} = \text{clamp}(C_{mixed}, B_{min}, B_{max}) $$
9.  **Return:** Output $C_{final}$ [S1, unofficial].

**Accumulation Step:**
After the reprojection pass, a separate accumulation shader stores the current frame's final color into `accumulation_attachment` for use in the next frame [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reprojection_pass(u, v, colour_attachment, position_attachment, accumulation_attachment, 
                      last_camera_matrix, reprojection_factor, indirect_lighting):
    # 1. Load current data
    C_curr = colour_attachment.sample(u, v)
    P_world = position_attachment.sample(u, v)
    
    # 2. Guard check
    if indirect_lighting == 0 or reprojection_factor <= 0.0:
        return C_curr
    
    # 3. Compute local bounds (min/max) using current pixel and 8 neighbors
    # Neighbors: sample0 (center), sample1-8 (surrounding 8 pixels)
    # 'Plus' pattern: center, up, down, left, right
    plus_indices = [0, 1, 3, 5, 7] # Example indices for cross pattern
    min_plus = min(colour_attachment.sample(u, v + dy, v + dx) for dy, dx in plus_offsets)
    max_plus = max(colour_attachment.sample(u, v + dy, v + dx) for dy, dx in plus_offsets)
    
    # 'Square' pattern: 3x3 block
    square_indices = range(9) # All 9 pixels in 3x3
    min_square = min(colour_attachment.sample(u, v + dy, v + dx) for dy in [-1,0,1] for dx in [-1,0,1])
    max_square = max(colour_attachment.sample(u, v + dy, v + dx) for dy in [-1,0,1] for dx in [-1,0,1])
    
    # Average of plus bounds and square bounds
    B_min = (min_plus + min_square) / 2.0
    B_max = (max_plus + max_square) / 2.0
    
    # 4. Reproject position
    # Transform world position to previous frame's clip space
    P_clip = last_camera_matrix * P_world
    
    # Convert to NDC
    NDC = P_clip.xyz / P_clip.w
    
    # Convert to texture coordinates
    T_proj = (NDC.xy + 1.0) / 2.0
    
    # 5. Validate projection
    if not (0.0 <= T_proj.x <= 1.0 and 0.0 <= T_proj.y <= 1.0):
        return C_curr # Invalid history, use current color
    
    # 6. Fetch history
    C_hist = accumulation_attachment.sample(T_proj.x, T_proj.y)
    
    # 7. Blend
    alpha = reprojection_factor
    C_mixed = C_curr * (1.0 - alpha) + C_hist * alpha
    
    # 8. Clamp to local bounds
    C_final = clamp(C_mixed, B_min, B_max)
    
    return C_final
```

**Parameters:**
*   `reprojection_factor`: Float uniform controlling the blend weight. Typical values are not specified in sources, but it must be $> 0.0$ for reprojection to occur [S1, unofficial].
*   `indirect_lighting`: Unsigned integer uniform. Reprojection is skipped if this is zero [S1, unofficial].

## Subtopics

### Alpha-Max Quasi-Depth Motion Vector
A method to compute screen-space motion vectors for direct volume rendering (DVR) by using the 3D position of the point with maximum alpha along the ray as a proxy for depth, enabling temporal reprojection despite the lack of a single surface depth value [S2]. Unlike standard reprojection which relies on a Z-buffer or explicit depth attachment, this variant identifies the point $p$ in world space where the alpha value is maximum along the ray passing through the volume. It then reprojects this world space position $p$ back to the previous frame's coordinate system using previous camera transformation matrices to obtain coordinates $(u', v')$. The screen-space motion vector is calculated as the difference between the current coordinates $(u, v)$ and the reprojected coordinates $(u', v')$ [S2]. This heuristic is chosen because the point of maximum alpha has the maximum contribution to the final accumulated sample, providing an acceptable approximation for motion estimation in volumetric data [S2].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per pixel for the reprojection pass, as it involves a constant number of texture lookups and arithmetic operations. The accumulation pass is also $O(1)$ per pixel.
*   **Memory Complexity:** Requires storage for the current color buffer, position buffer, and accumulation buffer. The accumulation buffer typically uses `rgba16float` to balance precision and memory usage [S1, unofficial].
*   **Performance:** Reduces per-pixel noise and flickering, allowing for lower sample counts or faster rendering times by leveraging temporal coherence [S1, unofficial][S2].
*   **Strengths:** Effective at stabilizing images over time, reducing high-frequency noise, and mitigating aliasing artifacts caused by camera motion [S1, unofficial][S2].
*   **Limitations:** Prone to ghosting artifacts if disocclusion is not handled correctly (e.g., if the projected position is invalid but history is still used) [S2]. Edge bleeding can occur if local bounds are not computed correctly [S1, unofficial].
*   **When to use:** When rendering dynamic volumes with camera motion where temporal stability is desired and computational budget for per-frame sampling is limited [S1, unofficial][S2].
*   **When not to use:** When rendering static scenes with no camera motion, or when high-fidelity edge preservation is critical and ghosting artifacts are unacceptable [S1, unofficial][S2].

## Background
Reprojection with Temporal Accumulation is a technique adapted from real-time rasterization (Temporal Anti-Aliasing) for use in voxel and volume rendering. It was implemented in the `bevy-voxel-engine` project to stabilize voxel rendering over time [S1, unofficial]. The concept of using temporal coherence to reduce noise in volume rendering has also been explored in academic contexts, such as in the context of super-resolution for medical volume visualization, where motion vectors are estimated using alpha-max quasi-depth for reprojection [S2].

## Sources
[S1] ria8651/bevy-voxel-engine, https://github.com/ria8651/bevy-voxel-engine, fetched 2026-10-27 (unofficial)
[S2] Deep Learning Based Super-Resolution for Medical Volume Visualization with Direct Volume Rendering, https://doi.org/10.1007/978-3-031-20713-6_8, fetched 2026-10-26
