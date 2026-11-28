# Depth Prepass with Equal-Depth Shading


## Overview
Optimizes rendering of overlapping opaque spheres by ensuring shading and ray tracing execute only for the nearest sphere per pixel, bypassing the loss of GPU hidden-surface removal caused by discard-based impostors [S1, unofficial]. It differs from standard depth testing by using a dedicated depth prepass followed by a shading pass with `depth_compare: Equal`, which restores per-pixel culling on architectures where fragment discard disables hardware optimizations [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of overlapping opaque primitives (spheres) [S1, unofficial].
*   **Output:** A rendered image where only the nearest primitive contributes to the final color per pixel [S1, unofficial].

**Data Structures:**
*   **Depth Buffer:** A standard depth attachment used in the prepass [S1, unofficial].
*   **Primitive Data:** Position $(x, y, z)$ and radius $r$ [S1, unofficial]. In the cited implementation, the radius share is stored in the $w$ component of the position vector [S1, unofficial].

**Steps:**
1.  **Depth Prepass:** Render the primitives to a depth buffer to record the depth of the nearest surface for each pixel using a dedicated shader (`room_spheres_depth.wgsl`) [S1, unofficial].
2.  **Shading Pass:** Render the primitives again to compute shading and ray tracing using a dedicated shader (`room_spheres_shade.wgsl`) [S1, unofficial]. Set the depth comparison function to `Equal` and disable depth writes (`depth_write: false`) [S1, unofficial]. This ensures the fragment shader runs only if the current primitive's depth matches the depth stored in the buffer from the prepass, effectively selecting the nearest primitive [S1, unofficial].
3.  **Depth Consistency:** Apply the `@invariant` qualifier to the clip-space position input in the shaders [S1, unofficial]. This ensures consistent depth rounding across the two separate rendering passes, preventing visual artifacts at primitive boundaries due to floating-point precision differences between passes or GPUs [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def run_depth_prepass_with_equal_depth_shading(primitives, projection_matrix, view_matrix):
    # Initialize depth buffer with maximum depth
    depth_buffer = [[float('inf') for _ in range(width)] for _ in range(height)]
    frame_buffer = [[None for _ in range(width)] for _ in range(height)]

    # Step 1: Depth Prepass
    # Render primitives to depth buffer with standard depth testing
    for prim in primitives:
        clip_pos = projection_matrix @ view_matrix @ vec4(prim.position, 1.0)
        pixel = clip_pos.xy
        if 0 <= pixel.x < width and 0 <= pixel.y < height:
            depth_buffer[pixel.y][pixel.x] = min(depth_buffer[pixel.y][pixel.x], clip_pos.z)

    # Step 2: Shading Pass
    # Render primitives again with Equal depth comparison and no depth writes
    for prim in primitives:
        clip_pos = projection_matrix @ view_matrix @ vec4(prim.position, 1.0)
        pixel = clip_pos.xy
        if 0 <= pixel.x < width and 0 <= pixel.y < height:
            # Depth test is set to 'Equal' in the render pipeline state
            if depth_buffer[pixel.y][pixel.x] == clip_pos.z:
                color = compute_shading_and_ray_tracing(prim)
                frame_buffer[pixel.y][pixel.x] = color
```

**Parameters:**
*   `depth_compare`: `Equal` [S1, unofficial].
*   `depth_write`: `false` in the shading pass [S1, unofficial].
*   `@invariant`: Applied to position input [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Apple GPUs:** Built-in hidden-surface removal is disabled when impostors use `discard`, causing every overlapping primitive to run full shading and ray tracing; the depth prepass restores per-pixel culling [S1, unofficial].
*   **Depth Consistency:** Without `@invariant`, the same comparison drops to 46 pixels (0.004%) where two overlapping spheres meet at the same 24-bit depth; the old draw kept the first sphere drawn, and the equal-depth pass keeps the last [S1, unofficial]. With the prepass also off, it drops to 3 pixels, by 1/255 [S1, unofficial].
*   **Silhouette Artifacts:** The `@invariant` qualifier may cause minor silhouette boundary shifts (single pixels on sphere silhouettes) due to different rounding in development vs. release builds, flipping some rim pixels between the sphere and what is behind it [S1, unofficial].
*   **Ordering:** The equal-depth pass keeps the last primitive drawn that matches the depth [S1, unofficial].

## Subtopics

### Depth Consistency Handling via @invariant
Uses the `@invariant` qualifier on clip-space positions to ensure consistent depth rounding across separate rendering passes, preventing visual artifacts at sphere boundaries [S1, unofficial]. It accepts minor silhouette boundary shifts (single pixels on sphere silhouettes) to guarantee depth consistency across pipelines on different GPUs [S1, unofficial]. Without it, the same `rt=on` comparison drops to 46 pixels (0.004%) where two overlapping spheres meet at the same 24-bit depth; the old draw kept the first sphere drawn there, and the equal-depth pass keeps the last [S1, unofficial]. With the prepass also off, it drops to 3 pixels, by 1/255 [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Requires an additional depth buffer attachment [S1, unofficial].
*   **Measured Performance:** The depth prepass for spheres followed by equal-depth shading reduces the main pass time by 27% [S1, unofficial]. Drawing walls after spheres was slower (4.7 ms vs 4.1 ms) because walls are opaque and the GPU's hidden-surface removal already skips them [S1, unofficial].
*   **Strengths:** Restores GPU hidden-surface removal on architectures where it is disabled by discard-based impostors and ensures shading and ray tracing run only for the nearest primitive [S1, unofficial].
*   **Limitations:** Requires two rendering passes and potential for minor visual artifacts at silhouettes due to depth rounding differences [S1, unofficial].
*   **When to Use:** When rendering overlapping opaque primitives that use discard-based impostors, especially on GPUs where hidden-surface removal is sensitive to discard [S1, unofficial].
*   **When Not to Use:** When primitives are not overlapping or when standard depth testing is sufficient (e.g., opaque walls, which are already skipped by GPU hidden-surface removal without needing a prepass) [S1, unofficial].

## Background
Developed as part of the `voxel-gi-particles` optimization effort to accelerate ray-traced lightbox rendering with voxel GI for overlapping particle systems [S1, unofficial]. It addresses performance issues on Apple GPUs where discard-based impostors broke hidden-surface removal [S1, unofficial]. It is part of a broader effort to optimize voxel cone tracing and particle rendering in real-time applications [S1, unofficial].

## Sources
[S1] voxel-gi-particles: ray-traced lightbox about twice as fast, same image, https://github.com/Siroko/kansei/pull/114, fetched 2026-10-26 (unofficial)
