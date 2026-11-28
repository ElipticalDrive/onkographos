# Gaussian Splat TAA Jitter Integration

Also called: gaussian_splatting: render splats with the engine's TAA jitter

## Overview
This method integrates temporal anti-aliasing (TAA) jitter into the Gaussian Splatting rendering pipeline to prevent splats from appearing "swimming" or bit-stable while surrounding meshes are correctly jittered. It solves the problem where splats, previously rendered with an unjittered projection matrix, failed to accumulate temporal history correctly under TAA and temporal upscaling (e.g., FSR2), causing residual displacement and visual artifacts. It differs from standard mesh rendering by explicitly excluding the engine's depth correction from the jittered projection, as the Gaussian pipeline relies on linear view-space depth.

## How it works
**Inputs:**
- `scene_data->cam_projection`: The base camera projection matrix.
- `scene_data->flip_y`: Boolean indicating if the Y-axis of the projection needs flipping.
- `scene_data->taa_jitter`: A 2D vector representing the temporal jitter offset. It is exactly $(0,0)$ unless a temporal stage (TAA or temporal upscaling) is active.
- `scene_data->depth_correction`: The engine's standard depth correction matrix (ignored by this method).

**Outputs:**
- `render_projection`: The GPU projection matrix, including jitter, used for culling and rendering.

**Data Structures:**
- `Projection`: A 4x4 matrix structure. Access via `columns[row][col]`.
- `Vector2`: A 2D vector with `x` and `y` components.

**Steps:**
1. **Initialize:** Copy `scene_data->cam_projection` into `render_projection`.
2. **Flip Y:** If `scene_data->flip_y` is true, negate the Y-axis scaling factor:
   $$ \text{render\_projection}[1][1] = -\text{render\_projection}[1][1] $$
3. **Check Jitter:** If `scene_data->taa_jitter` is exactly $(0,0)$, return `render_projection` immediately. This short-circuits the non-temporal path, ensuring bit-identical results to the pre-jitter state.
4. **Apply Jitter:** If `scene_data->taa_jitter` is non-zero, apply an identity-plus-translation correction to `render_projection`. This matches the engine's standard jitter application method (e.g., `Projection::add_jitter_offset()`).
   - The correction is a translation in the projection space corresponding to the jitter vector.
   - **Crucial:** Do **not** apply the engine's depth correction. The Gaussian pipeline uses linear view-space depth, so the depth correction would distort the splat positions.
5. **Return:** Return the modified `render_projection`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_render_projection(cam_projection, flip_y, taa_jitter):
    """
    Builds the GPU projection matrix for Gaussian Splatting with TAA jitter.
    
    Args:
        cam_projection (Matrix4x4): Base camera projection.
        flip_y (bool): Whether to flip the Y-axis.
        taa_jitter (Vector2): Temporal jitter offset (0,0) if no temporal stage.
        
    Returns:
        Matrix4x4: The jittered projection matrix.
    """
    # Step 1: Copy base projection
    render_projection = cam_projection.copy()
    
    # Step 2: Apply Y-flip if necessary
    if flip_y:
        # Negate the Y-axis scaling factor (row 1, col 1 in column-major or row-major depending on engine, 
        # but typically [1][1] refers to the Y-scale component in the projection matrix)
        render_projection[1][1] = -render_projection[1][1]
        
    # Step 3: Check for zero jitter
    if taa_jitter.x == 0.0 and taa_jitter.y == 0.0:
        return render_projection
        
    # Step 4: Apply TAA jitter
    # The engine's standard jitter application is an identity-plus-translation correction.
    # This modifies the projection matrix to offset the clip-space coordinates.
    # Note: We do NOT apply depth correction here.
    render_projection = add_jitter_offset(render_projection, taa_jitter)
    
    return render_projection

def add_jitter_offset(projection, jitter):
    """
    Applies an identity-plus-translation correction to the projection matrix.
    This is a standard engine function that modifies the projection matrix
    to account for temporal jitter.
    """
    # Implementation details of add_jitter_offset are engine-specific,
    # but conceptually it adds the jitter vector to the projection's
    # translation components in a way that affects clip-space coordinates.
    # For Gaussian Splatting, this is the only modification needed.
    # [G] Assuming standard engine behavior: the jitter is applied to the
    # projection matrix's last column or via a specific helper that
    # adjusts the x and y offsets in clip space.
    # Since the source cites `Projection::add_jitter_offset()`, we assume
    # this function exists and performs the correct matrix modification.
    return projection.apply_jitter(jitter)
```

**Parameters:**
- `scene_data->taa_jitter`: A vector representing the temporal jitter offset. It is exactly $(0,0)$ unless a temporal stage (TAA or temporal upscaling) is active.

**Edge Cases and Pitfalls:**
- **Cache Key:** The render cache reuse key must include the jittered projection matrix. Previously, the key was the unjittered `p_input.projection`, causing static camera frames to re-compose frozen jitter phases every frame. The key must be the GPU projection carrying the jitter.
- **Depth Correction:** Do not apply the engine's depth correction to the projection. The Gaussian pipeline uses linear view-space depth, and applying depth correction would distort the splat positions.
- **Zero Jitter:** If `taa_jitter` is zero, the method must short-circuit and return the flipped matrix immediately without further modification to ensure bit-identical results to the pre-jitter state.

## Operating characteristics
- **Time Complexity:** $O(1)$ for matrix modification. The short-circuit for zero jitter ensures minimal overhead when no temporal stage is active.
- **Memory Complexity:** $O(1)$ additional memory for the projection matrix.
- **Measured Performance:**
  - **TAA:** Under TAA, splats previously had 0.0000 px displacement (bit-stable). After the fix, splats move by ~0.0156 px, matching the mesh order of magnitude (~0.0152 px).
  - **FSR2 @1.0:** Splat displacement reduced from 0.2525 px to 0.0900 px (-64%).
  - **FSR2 @0.5:** Splat displacement reduced from 0.3603 px to 0.0927 px (-74%).
- **Strengths:** Eliminates "swimming" artifacts under TAA and temporal upscaling. Matches mesh jitter behavior.
- **Limitations:** Residual displacement of ~0.09 px remains under FSR2 due to FSR2 not accumulating history on splat pixels (clamped to 0.9 destination alpha). This is not a defect of the jitter integration but a limitation of the composite feed.
- **When to Use:** Use when rendering Gaussian Splats in a scene with TAA or temporal upscaling to ensure visual consistency with meshes.
- **When Not to Use:** Not applicable; this is a necessary fix for correct temporal behavior.

## Background
This method originated from a fix for the Gaussian Splat Renderer in Godot, where splats were not jittered, causing them to 'swim' under FSR2 and remain bit-stable (non-antialiased) under TAA, while meshes were correctly jittered. The fix was implemented in PR #929 and #1026 by klausi3D in the `godotGS` repository. The issue was identified as the projection uploaded to the splat GPU pipeline never carrying the engine's temporal jitter. The fix involved modifying `GaussianSplatRenderer::build_render_projection()` to apply the jitter and updating the render cache key to include the jittered projection.

## Sources
[S1] gaussian_splatting: render splats with the engine's TAA jitter, and key the render cache on the matrix that carries it (#929), https://github.com/klausi3D/godotGS/pull/1026, fetched 2026-10-28 (unofficial)
