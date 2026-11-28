# Temporal Accumulation (Closest Hit)

## Overview
Temporal Accumulation (Closest Hit) stabilizes stochastic or noisy direct volume ray casting results by blending the current frame with a history buffer of previous frames [S2, S3, S4, unofficial]. Unlike simple frame averaging, this method specifically selects the "closest hit" (the sample with the smallest depth/opacity weight) between the current frame and the accumulated history to maintain geometric stability and reduce ghosting artifacts during camera movement [S3, S5, unofficial].

## How it works
The method executes as a multi-pass rendering pipeline or fragment shader logic block that manages a history buffer [S1, S4, unofficial]. It requires the current frame's color/depth and the previous frame's reprojection data [S4, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   `current_frame`: Current render pass output (color and depth/opacity) [S3, S5, unofficial].
    *   `current_depth`: Depth value for the current pixel [S4, unofficial].
    *   `history_buffer`: Accumulated data from previous frames [S1, S4, unofficial].
    *   `camera_matrices`: Current and previous View-Projection matrices ($V_{curr}, P_{curr}, V_{prev}, P_{prev}$) [S4, unofficial].
    *   `camera_position`: World-space camera origin [S4, unofficial].
*   **Outputs**:
    *   `accumulated_frame`: The final stabilized color buffer [S1, S4, unofficial].

### Data Structures
The history is typically stored in a texture array or a circular buffer to save memory [S1, S2, unofficial].

1.  **History Texture Array** (`history_frames`):
    *   Dimensions: $\{width, height, N\_HISTORY\_ACC\}$ [S1, unofficial].
    *   Format: RGBA32Float or equivalent surface format [S1, unofficial].
    *   Usage: `RENDER_ATTACHMENT | TEXTURE_BINDING` [S1, unofficial].
    *   Indexing: Layer index is `frame_count % N_HISTORY_ACC` [S1, unofficial].
2.  **Normal/Depth Texture** (`normal_depth_texture`):
    *   Dimensions: $\{width, height, 1\}$ [S1, unofficial].
    *   Format: RGBA32Float [S1, unofficial].
    *   Stores depth and potentially normal/opacity info for reprojection [S1, unofficial].
3.  **Uniforms** (for shader passes):
    *   `uCurrent`: Sampler for current frame [S4, unofficial].
    *   `uCurrentDepth`: Sampler for current depth [S4, unofficial].
    *   `uHistory`: Sampler for history buffer [S4, unofficial].
    *   `uInvViewProj`: Inverse of current View-Projection matrix [S4, unofficial].
    *   `uPrevViewProj`: Inverse of previous View-Projection matrix [S4, unofficial].
    *   `uCamPos`: Camera world position [S4, unofficial].
    *   `uBlend`: Blending weight (0.0 to 1.0) [S4, unofficial].
    *   `uHistoryValid`: Boolean flag (0 or 1) indicating if history is valid [S4, unofficial].
    *   `uInvFrameNumber`: Inverse of current frame count (for running average) [S3, S5, unofficial].
    *   `uMix`: Mixing factor for accumulator/frame blend [S5, unofficial].

### Steps

1.  **Initialization (Closest Hit Texture Initialization)**:
    *   Reset the accumulation buffer to a value indicating "no hit" (e.g., `vec4(-1)`) when the scene changes or at startup [S3, unofficial].
    *   Clear `uHistoryValid` to 0 [S4, unofficial].

2.  **Ray Pass (Current Frame Generation)**:
    *   Render the volume using standard ray casting [S1, unofficial].
    *   Output the result to `history_frames` at layer `frame_count % N_HISTORY_ACC` and to `normal_depth_texture` [S1, unofficial].
    *   Store the current frame's depth/opacity in the `w` component of the color vector for closest-hit comparison [S3, S5, unofficial].

3.  **Reprojection and Blending (Temporal Pass)**:
    *   **Check Validity**: If `uHistoryValid == 0`, output `current_frame` directly and set `uHistoryValid = 1` [S4, unofficial].
    *   **World Position Reconstruction**:
        *   Compute ray direction in world space from NDC coordinates $(u, v)$:
            $$
            \mathbf{ndc} = (2u - 1, 2v - 1)
            $$
            $$
            \mathbf{p}_{near} = V_{curr}^{-1} P_{curr}^{-1} (\mathbf{ndc}_x, \mathbf{ndc}_y, -1, 1)
            $$
            $$
            \mathbf{p}_{far} = V_{curr}^{-1} P_{curr}^{-1} (\mathbf{ndc}_x, \mathbf{ndc}_y, 1, 1)
            $$
            $$
            \mathbf{rd} = \text{normalize}\left(\frac{\mathbf{p}_{far}.xyz}{\mathbf{p}_{far}.w} - \frac{\mathbf{p}_{near}.xyz}{\mathbf{p}_{near}.w}\right)
            $$
        *   Fetch depth $d$ from `uCurrentDepth` [S4, unofficial].
        *   Compute world position $\mathbf{worldPos}$:
            $$
            \mathbf{worldPos} = \mathbf{uCamPos} + \mathbf{rd} \cdot (d > 0 ? d : 200000.0)
            $$
            *Pitfall*: Pixels with no cloud ($d \le 0$) cause large parallax errors during reprojection; substituting a far distance (200000.0) handles this [S4, unofficial].
    *   **Project to Previous Frame**:
        *   Transform $\mathbf{worldPos}$ to previous clip space:
            $$
            \mathbf{prevClip} = V_{prev} P_{prev} (\mathbf{worldPos}, 1)
            $$
        *   Compute previous UV:
            $$
            \mathbf{prevUv} = \left(\frac{\mathbf{prevClip}.xy}{\mathbf{prevClip}.w}\right) \cdot 0.5 + 0.5
            $$
    *   **Boundary Check**:
        *   Evaluate screen validity: `onScreen = prevClip.w > 0.0 && all(greaterThanEqual(prevUv, vec2(0.0))) && all(lessThanEqual(prevUv, vec2(1.0)))` [S4, unofficial].
        *   If `!onScreen` (i.e., `prevClip.w <= 0.0` or `prevUv` outside `[0, 1]`), output `current_frame` [S4, unofficial].
    *   **Closest Hit Selection**:
        *   Sample current frame: $\mathbf{frame} = \text{texture}(\mathbf{uCurrent}, \mathbf{vUv})$ [S3, S5, unofficial].
        *   Sample history: $\mathbf{acc} = \text{texture}(\mathbf{uHistory}, \mathbf{prevUv})$ [S3, S5, unofficial].
        *   Let $w_{frame}$ and $w_{acc}$ be the opacity/depth weights (typically the $w$ component) [S3, S5, unofficial].
        *   If $w_{frame} > 0$ and $w_{acc} > 0$:
            $$
            \mathbf{oClosest} = (w_{frame} < w_{acc}) ? \mathbf{frame} : \mathbf{acc}
            $$
        *   If only $w_{frame} > 0$: $\mathbf{oClosest} = \mathbf{frame}$ [S3, unofficial].
        *   If only $w_{acc} > 0$: $\mathbf{oClosest} = \mathbf{acc}$ [S3, unofficial].
        *   If neither: $\mathbf{oClosest} = \mathbf{acc}$ [S3, unofficial].
    *   **Noise Clamping and Blending**:
        *   Compute 3x3 neighborhood min/max of current sample to bound noise [S4, unofficial]:
            $$
            \mathbf{lo} = \min_{x,y \in \{-1,0,1\}} \text{texture}(\mathbf{uCurrent}, \mathbf{vUv} + (x,y)\Delta\mathbf{uv})
            $$
            $$
            \mathbf{hi} = \max_{x,y \in \{-1,0,1\}} \text{texture}(\mathbf{uCurrent}, \mathbf{vUv} + (x,y)\Delta\mathbf{uv})
            $$
        *   Pad bounds:
            $$
            \mathbf{lo}' = \mathbf{lo} - (\mathbf{hi} - \mathbf{lo}) \cdot 0.25 - 0.002
            $$
            $$
            \mathbf{hi}' = \mathbf{hi} + (\mathbf{hi} - \mathbf{lo}) \cdot 0.25 + 0.002
            $$
        *   Clamp history sample:
            $$
            \mathbf{hist} = \text{clamp}(\mathbf{acc}, \mathbf{lo}', \mathbf{hi}')
            $$
        *   Blend:
            $$
            \mathbf{oColor} = \text{mix}(\mathbf{hist}, \mathbf{oClosest}, \mathbf{uBlend})
            $$
            *Alternative*: Use running average: $\mathbf{oColor} = \mathbf{acc} + (\mathbf{frame} - \mathbf{acc}) \cdot \mathbf{uInvFrameNumber}$ [S3, S5, unofficial].
            *Alternative*: Use mixing factor: $\mathbf{oColor} = \text{mix}(\mathbf{acc}, \mathbf{frame}, \mathbf{uMix})$ [S5, unofficial].

4.  **Final Output**:
    *   Present the accumulated frame to the screen [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def temporal_accumulation_closest_hit(current_frame_tex, current_frame_uv, current_depth,
                                      history_buffer, u_inv_view_proj, prev_view_proj, cam_pos,
                                      u_blend, u_history_valid, frame_count, n_history_acc):
    # 1. Handle invalid history or first frame
    if frame_count == 0 or u_history_valid == 0:
        return current_frame_tex[current_frame_uv]

    # 2. Reconstruct world position from current depth
    ndc = vec2(current_frame_uv.x * 2.0 - 1.0, current_frame_uv.y * 2.0 - 1.0)
    p_near = u_inv_view_proj * vec4(ndc.x, ndc.y, -1.0, 1.0)
    p_far = u_inv_view_proj * vec4(ndc.x, ndc.y, 1.0, 1.0)
    ray_dir = normalize(p_far.xyz / p_far.w - p_near.xyz / p_near.w)

    depth = current_depth
    if depth <= 0.0:
        depth = 200000.0

    world_pos = cam_pos + ray_dir * depth

    # 3. Project to previous frame UV
    prev_clip = prev_view_proj * vec4(world_pos, 1.0)
    prev_uv = (prev_clip.xy / prev_clip.w) * 0.5 + 0.5

    # 4. Check bounds
    on_screen = prev_clip.w > 0.0 and 0.0 <= prev_uv.x <= 1.0 and 0.0 <= prev_uv.y <= 1.0
    if not on_screen:
        return current_frame_tex[current_frame_uv]

    # 5. Sample history
    history_sample = texture(history_buffer, prev_uv)

    # 6. Closest Hit Selection
    w_frame = current_frame_tex[current_frame_uv].w
    w_hist = history_sample.w

    if w_frame > 0.0 and w_hist > 0.0:
        closest = current_frame_tex[current_frame_uv] if w_frame < w_hist else history_sample
    elif w_frame > 0.0:
        closest = current_frame_tex[current_frame_uv]
    elif w_hist > 0.0:
        closest = history_sample
    else:
        closest = history_sample

    # 7. Noise Clamping
    texel_size = vec2(1.0 / current_frame_tex.width, 1.0 / current_frame_tex.height)
    lo = current_frame_tex[current_frame_uv]
    hi = current_frame_tex[current_frame_uv]
    for y in [-1, 0, 1]:
        for x in [-1, 0, 1]:
            neighbor = texture(current_frame_tex, current_frame_uv + vec2(x, y) * texel_size)
            lo = min(lo, neighbor)
            hi = max(hi, neighbor)

    pad = (hi - lo) * 0.25 + 0.002
    lo_clamped = lo - pad
    hi_clamped = hi + pad
    hist_clamped = max(lo_clamped, min(hi_clamped, history_sample))

    # 8. Blend
    result = mix(hist_clamped, closest, u_blend)
    return result
```

### Parameters
*   `N_HISTORY_ACC`: Number of history frames stored. Must be $>0$ [S1, unofficial].
*   `uBlend`: Interpolation factor between clamped history and current closest hit [S4, unofficial].
*   `uMix`: Blending factor between accumulator and frame [S5, unofficial].
*   `uInvFrameNumber`: Inverse of current frame number for running average [S3, S5, unofficial].
*   `uHistoryValid`: Flag to skip reprojection on first frame or scene change [S4, unofficial].

### Edge Cases and Pitfalls
*   **Disocclusion/Ghosting**: Caused by camera movement or object animation. Handled by clamping history samples to the padded min/max neighborhood of the current frame [S4, unofficial].
*   **Empty Pixels**: Depth $\le 0$ causes large parallax errors. Substitute a far distance (e.g., 200000.0) [S4, unofficial].
*   **Buffer Overflow**: If `frame_count` overflows or `N_HISTORY_ACC` is 0, layer indexing fails. Use modulo arithmetic `frame_count % N_HISTORY_ACC` [S1, unofficial].
*   **Scene Changes**: Camera/object transformations invalidate history. Clear the buffer and reset `uHistoryValid` [S2, S4, unofficial].
*   **Memory Limits**: Multi-frame buffers require dynamic memory modification, which is limited. Single-buffer overwrite (circular buffer) is preferred [S2, unofficial].

## Subtopics

### Closest Hit Texture Initialization
A fragment shader pass that initializes the closest hit texture to a value indicating no hit (e.g., `vec4(-1)`) [S3, unofficial]. This resets the accumulation buffer when scene conditions change or at startup [S3, unofficial].

### Accumulation Integration
A rendering pass that blends an accumulated buffer with a new frame using a mixing factor [S5, unofficial]. It samples the accumulator and current frame, selects the closest depth/position (smaller `w` value) if both are valid, and blends them using `uMix` [S5, unofficial].

### Running Average Integration
A technique to combine frames over time to reduce noise, using an exponential moving average or simple frame blending based on frame count [S3, unofficial]. It updates the accumulator as `acc + (frame - acc) * uInvFrameNumber` [S3, S5, unofficial].

### Temporal Layer Averaging
A fragment shader pass that averages color values across multiple layers of a 2D texture array [S1, unofficial]. It loops through all layers, sums the values, and divides by the number of layers [S1, unofficial]. Used for temporal accumulation or history blending [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(1)$ per pixel per frame, assuming constant number of history layers and fixed neighborhood size for clamping [S1, S4, unofficial].
*   **Memory Complexity**: $O(W \times H \times N\_HISTORY\_ACC)$ for the history buffer [S1, unofficial]. Single-buffer overwrite reduces this to $O(W \times H)$ if only the immediate previous frame is stored [S2, unofficial].
*   **Performance**: Significantly reduces noise in stochastic ray casting, allowing for fewer samples per frame [S2, S4, unofficial].
*   **Strengths**: Stabilizes flickering, reduces noise, maintains sharp edges via closest-hit selection [S3, S4, unofficial].
*   **Limitations**: Ghosting artifacts during rapid motion or disocclusion if clamping is insufficient [S4, unofficial]. Requires reprojection matrices and depth buffers [S4, unofficial].

## Background
Temporal accumulation has been used in volume rendering to stabilize stochastic results, such as those from sparse voxel octrees or cloud rendering [S2, S4, unofficial]. The "Closest Hit" variant specifically addresses the issue of selecting the most relevant historical data to prevent ghosting of occluded geometry [S3, S5, unofficial]. Implementations vary from simple frame blending to complex reprojection with noise clamping [S1, S4, unofficial].

## Sources
[S1] newDINO/voxel_ray_demo, https://github.com/newDINO/voxel_ray_demo, fetched 2026-10-24 (unofficial)
[S2] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
[S3] rokcej/volumetric-ray-casting, https://github.com/rokcej/volumetric-ray-casting, fetched 2026-10-26 (unofficial)
[S4] harkawataryan/Volumetric-Cloud-Renderer, https://github.com/harkawataryan/Volumetric-Cloud-Renderer, fetched 2026-10-27 (unofficial)
[S5] terier/vpt, https://github.com/terier/vpt, fetched 2026-10-26 (unofficial)
