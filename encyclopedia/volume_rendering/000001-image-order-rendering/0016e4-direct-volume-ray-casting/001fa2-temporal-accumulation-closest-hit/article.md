# Temporal Accumulation (Closest Hit)

Also called: Closest Hit Texture Initialization, Accumulation Integration, Running Average Integration, Temporal Layer Averaging

## Overview
Temporal Accumulation (Closest Hit) stabilizes stochastic or noisy direct volume ray casting results by blending the current frame with a history buffer of previous frames [S2, S3, S4, unofficial]. Unlike simple frame averaging, this method specifically selects the "closest hit" (the sample with the smallest depth/opacity weight) between the current frame and the accumulated history to maintain geometric stability and reduce ghosting artifacts during camera movement [S3, S5, unofficial]. It builds on [Direct volume ray casting](19-direct-volume-ray-casting.md) by adding a post-processing or integrated accumulation pass that reprojects previous frame data into the current view space.

## How it works
The method operates as a multi-pass rendering pipeline or a fragment shader logic block that manages a history buffer. It requires the current frame's color/depth and the previous frame's reprojection data.

### Inputs and Outputs
*   **Inputs**:
    *   `current_frame`: Current render pass output (color and depth).
    *   `history_buffer`: Accumulated data from previous frames.
    *   `camera_matrices`: Current and previous View-Projection matrices ($V_{curr}, P_{curr}, V_{prev}, P_{prev}$).
    *   `camera_position`: World-space camera origin.
*   **Outputs**:
    *   `accumulated_frame`: The final stabilized color buffer.

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
    *   *Note*: The current frame's depth/opacity is stored in the `w` component of the color vector for closest-hit comparison [S3, S5, unofficial].

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
            *Pitfall*: Pixels with no cloud ($d \le 0$) cause large parallax errors; substitute a far distance (e.g., 200000.0) [S4, unofficial].
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
        *   If $\mathbf{prevClip}.w \le 0$ or $\mathbf{prevUv}$ is outside $[0, 1]$, output `current_frame` [S4, unofficial].
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
        *   If neither: $\mathbf{oClosest} = \mathbf{acc}$ (or black) [S3, unofficial].
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
def temporal_accumulation_closest_hit(current_frame, current_depth, history_buffer, 
                                      prev_view_proj, cam_pos, u_blend, u_inv_frame_number, 
                                      u_mix, frame_count, n_history_acc):
    """
    Performs temporal accumulation with closest hit selection.
    
    Args:
        current_frame: vec4 (color/depth/opacity) for current pixel
        current_depth: float depth value for current pixel
        history_buffer: 2D array or texture of previous frames
        prev_view_proj: mat4 (previous view * projection)
        cam_pos: vec3 (camera world position)
        u_blend: float (blending weight)
        u_inv_frame_number: float (1.0 / frame_count)
        u_mix: float (mixing factor)
        frame_count: int (current frame index)
        n_history_acc: int (number of history layers)
        
    Returns:
        vec4: Accumulated color
    """
    
    # 1. Handle invalid history or first frame
    if frame_count == 0 or not history_buffer.is_valid():
        return current_frame
    
    # 2. Reconstruct world position from current depth
    # Assuming NDC to World conversion logic is available via helper
    ndc = vec2(current_frame.uv.x * 2.0 - 1.0, current_frame.uv.y * 2.0 - 1.0)
    
    # Get ray direction in world space (simplified, assumes perspective projection)
    # In practice, this requires inverse view-projection matrices
    ray_dir = compute_world_ray_direction(ndc, current_frame.uv)
    
    depth = current_depth
    if depth <= 0.0:
        depth = 200000.0 # Far plane substitute for empty pixels
    
    world_pos = cam_pos + ray_dir * depth
    
    # 3. Project to previous frame UV
    prev_clip = prev_view_proj * vec4(world_pos, 1.0)
    prev_uv = (prev_clip.xy / prev_clip.w) * 0.5 + 0.5
    
    # 4. Check bounds
    if prev_clip.w <= 0.0 or not (0.0 <= prev_uv.x <= 1.0 and 0.0 <= prev_uv.y <= 1.0):
        return current_frame
    
    # 5. Sample history
    history_sample = history_buffer.sample(prev_uv)
    
    # 6. Closest Hit Selection
    # Assume w component stores opacity/depth weight
    w_frame = current_frame.w
    w_hist = history_sample.w
    
    if w_frame > 0.0 and w_hist > 0.0:
        if w_frame < w_hist:
            closest = current_frame
        else:
            closest = history_sample
    elif w_frame > 0.0:
        closest = current_frame
    elif w_hist > 0.0:
        closest = history_sample
    else:
        closest = history_sample # Fallback
    
    # 7. Noise Clamping (Optional but recommended for stochastic rays)
    # Compute 3x3 neighborhood min/max of current frame
    texel_size = vec2(1.0 / current_frame.texture_width, 1.0 / current_frame.texture_height)
    lo = current_frame
    hi = current_frame
    for y in [-1, 0, 1]:
        for x in [-1, 0, 1]:
            neighbor = current_frame.sample(current_frame.uv + vec2(x, y) * texel_size)
            lo = min(lo, neighbor)
            hi = max(hi, neighbor)
            
    # Pad bounds
    pad = (hi - lo) * 0.25 + 0.002
    lo_clamped = lo - pad
    hi_clamped = hi + pad
    
    # Clamp history sample
    hist_clamped = clamp(history_sample, lo_clamped, hi_clamped)
    
    # 8. Blend
    # Method A: Weighted blend with clamped history
    result = mix(hist_clamped, closest, u_blend)
    
    # Method B: Running Average (if u_blend is not used)
    # result = history_sample + (current_frame - history_sample) * u_inv_frame_number
    
    # Method C: Simple Mix (if u_mix is used)
    # result = mix(history_sample, current_frame, u_mix)
    
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
A fragment shader pass that initializes the closest hit texture to a value indicating no hit (e.g., `vec4(-1)`). This resets the accumulation buffer when scene conditions change or at startup [S3, unofficial].

### Accumulation Integration
A rendering pass that blends an accumulated buffer with a new frame using a mixing factor. It samples the accumulator and current frame, selects the closest depth/position (smaller `w` value) if both are valid, and blends them using `uMix` [S5, unofficial].

### Running Average Integration
A technique to compute a running average of frames using the inverse frame number. It updates the accumulator as `acc + (frame - acc) * uInvFrameNumber` [S5, unofficial].

### Temporal Layer Averaging
A fragment shader pass that averages color values across multiple layers of a 2D texture array. It loops through all layers, sums the values, and divides by the number of layers. Used for temporal accumulation or history blending [S1, unofficial].

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
