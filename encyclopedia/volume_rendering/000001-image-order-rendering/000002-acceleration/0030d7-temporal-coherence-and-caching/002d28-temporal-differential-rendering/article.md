# Temporal Differential Rendering

Also called: Dirty-brick partial re-render

## Overview
Temporal differential rendering reduces computational load by skipping the rendering of screen tiles that correspond to unchanged regions of the volume, leveraging the assumption that most of the scene remains static between frames [S1, unofficial]. It differs from general temporal accumulation methods by explicitly tracking "dirty" data structures (such as physics-updated bricks) on the CPU and projecting them into screen-space tiles to determine which GPU compute passes can be entirely skipped or early-terminated [S1, unofficial].

## How it works
The method operates in two main phases: a CPU-side dirty-tile tracking phase and a GPU-side conditional rendering phase.

**Inputs:**
- Volume data structure (e.g., voxel grid or brick hierarchy).
- List of "dirty" bricks: indices of bricks modified by physics, editing, or loading [S1, unofficial].
- Previous frame's render state (optional, for early-out logic) [S1, unofficial].

**Outputs:**
- Final color buffer for the current frame.
- Updated dirty-tile mask for the next frame.

**Data Structures:**
1.  **Dirty Brick List:** A list of brick indices that have changed since the last frame [S1, unofficial].
2.  **Tile Mask:** A 2D array or texture representing the screen divided into $8 \times 8$ pixel tiles. Each element indicates whether the tile contains any dirty brick [S1, unofficial].
3.  **First-Hit Distance Texture:** An `r32float` texture storing the distance to the first hit for each tile, computed at a reduced resolution (1/8th of screen resolution, i.e., one value per $8 \times 8$ block) [S1, unofficial].

**Algorithm Steps:**

1.  **Identify Dirty Bricks:** Collect all bricks that have been modified by simulation or user interaction [S1, unofficial].
2.  **Project to Screen Tiles (CPU):**
    - For each dirty brick, compute its bounding box in screen space.
    - Mark all $8 \times 8$ pixel tiles that intersect with any dirty brick's screen-space projection as "dirty" in the Tile Mask [S1, unofficial].
    - If no dirty bricks exist, the Tile Mask is all zeros (clean) [S1, unofficial].
3.  **Check for Global Clean State:**
    - If the Tile Mask is entirely clean (no dirty tiles), skip both the pre-pass and main pass entirely [S1, unofficial].
4.  **Pre-Pass (Dirty Tile Detection):**
    - Launch a compute shader at 1/8 resolution (one ray per $8 \times 8$ tile) [S1, unofficial].
    - Each ray walks the volume hierarchy to find the first hit distance $d_{hit}$ [S1, unofficial].
    - Write $d_{hit}$ to the First-Hit Distance Texture [S1, unofficial].
5.  **Main Pass (Selective Rendering):**
    - Launch the main rendering compute shader [S1, unofficial].
    - For each tile:
        - Read the Tile Mask. If the tile is clean, **early-out** (skip rendering for this tile) [S1, unofficial].
        - If the tile is dirty, read the pre-pass first-hit distance $d_{hit}$ [S1, unofficial].
        - Fast-forward the ray to depth $d_{hit} - \delta$, where $\delta$ is a safety margin (e.g., 16 voxels) to ensure the ray starts inside or just before the relevant geometry [S1, unofficial].
        - Perform full volume rendering from this starting depth [S1, unofficial].
6.  **Distance LOD (Optional Optimization):**
    - For rays traveling beyond a threshold (e.g., 400 voxels), terminate DDA at brick granularity and shade with the brick's topmost solid material to maintain surface color for distant terrain [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# CPU Side
def update_dirty_tiles(volume, dirty_bricks, screen_width, screen_height, tile_size=8):
    """
    Projects dirty bricks into screen tiles.
    Returns a boolean mask of dirty tiles.
    """
    num_tiles_x = (screen_width + tile_size - 1) // tile_size
    num_tiles_y = (screen_height + tile_size - 1) // tile_size
    dirty_tile_mask = [[False] * num_tiles_x for _ in range(num_tiles_y)]
    
    for brick in dirty_bricks:
        # Project brick bounding box to screen space
        # This function depends on the specific projection matrix and brick geometry
        tile_bounds = project_brick_to_tiles(brick, screen_width, screen_height)
        for (ty, tx) in tile_bounds:
            if 0 <= ty < num_tiles_y and 0 <= tx < num_tiles_x:
                dirty_tile_mask[ty][tx] = True
                
    return dirty_tile_mask

# GPU Side (Compute Shader Pseudocode)
# Input: dirty_tile_mask, first_hit_texture, volume_data
# Output: color_buffer

@compute_shader
def render_frame(tile_idx_x, tile_idx_y):
    # Check if this tile is dirty
    if not dirty_tile_mask[tile_idx_y][tile_idx_x]:
        return # Early-out: skip rendering for this tile

    # Pre-pass result: first hit distance for this tile
    d_hit = first_hit_texture[tile_idx_y][tile_idx_x]
    
    # Safety margin
    safety_margin = 16 # voxels
    
    # Fast-forward ray
    ray_origin = camera_origin
    ray_direction = get_ray_direction(tile_idx_x, tile_idx_y)
    
    # Start rendering from d_hit - safety_margin
    start_depth = max(0.0, d_hit - safety_margin)
    
    # Perform volume rendering from start_depth
    color = volume_render_ray(ray_origin, ray_direction, start_depth, volume_data)
    
    # Write to color buffer
    color_buffer[tile_idx_y * num_tiles_x + tile_idx_x] = color
```

**Parameters:**
- `tile_size`: Size of screen tiles in pixels. Typical value: $8 \times 8$ [S1, unofficial].
- `safety_margin`: Distance in voxels to subtract from the first-hit distance to ensure the ray starts inside the relevant volume region. Typical value: 16 voxels [S1, unofficial].
- `LOD_threshold`: Distance beyond which DDA terminates at brick granularity. Typical value: 400 voxels [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** The first-hit distance texture uses `r32float`. Ensure sufficient precision for large scenes to avoid artifacts in ray fast-forwarding [S1, unofficial].
- **Boundary Conditions:** If a dirty brick is partially outside the view frustum, it must still be correctly projected to avoid missing dirty tiles [S1, unofficial].
- **Degenerate Input:** If no dirty bricks are present, the entire frame is skipped. This is a valid optimization but requires careful handling of frame timing and synchronization [S1, unofficial].

## Subtopics

### Partial Volume Rendering Update
A variant where a partial volume rendering pass is performed between passes of a visibility detection algorithm to complete the image and update occluding data with newly loaded bricks [S2, unofficial]. The frequency of total scene rendering must be chosen carefully due to its high cost [S2, unofficial]. A potential future extension involves progressive rendering via ray-tracing resumption, storing ray state (position, direction, hierarchy position, DDA stack state) in image-space buffers alongside color and opacity [S2, unofficial].

### Anti-flicker via Freezing
A heuristic used in differentiable rendering to prevent flicker in static regions by freezing primitives whose bounding boxes do not intersect an inter-frame difference mask [S3, unofficial]. The difference mask $D$ is computed as $D := \mathbb{1}(I_{target}^f \neq I_{target}^{f-1})$ [S3, unofficial]. A primitive $i$ is frozen if $B_i \cap \text{support}(D) = \emptyset$ [S3, unofficial]. This prevents spurious updates in static areas that perturb the loss and trigger compensatory updates in neighboring primitives [S3, unofficial].

## Operating characteristics
- **Time Complexity:** Proportional to the number of dirty tiles rather than the total number of pixels or volume elements. In static scenes, complexity approaches $O(1)$ as no rendering occurs [S1, unofficial].
- **Memory Complexity:** Requires storage for the dirty tile mask and the first-hit distance texture. The first-hit distance texture is at 1/8 resolution, reducing memory usage by a factor of 64 compared to full-resolution depth buffers [S1, unofficial].
- **Performance:** Significant speedup in scenes with low temporal change. Performance degrades as the number of dirty tiles increases, approaching the cost of full rendering when the entire screen is dirty [S1, unofficial].
- **Strengths:** Efficiently handles dynamic volumes with localized changes (e.g., physics simulations) [S1, unofficial].
- **Limitations:** Requires CPU-GPU synchronization to update the dirty tile mask, which can introduce latency. Not effective for scenes with global or widespread changes [S1, unofficial].

## Background
Temporal differential rendering was introduced in the context of voxel-based rendering systems, specifically in the `voxelG` project, to optimize rendering of dynamic voxel environments [S1, unofficial]. It builds on the principle of temporal coherence by explicitly tracking data changes at the brick level and mapping them to screen-space tiles to avoid redundant computation [S1, unofficial].

## Sources
[S1] mstampfli/voxelG, https://github.com/mstampfli/voxelG, fetched 2026-10-24 (unofficial)
[S2] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
[S3] DiffBMP: Differentiable Rendering with Bitmap Primitives, https://doi.org/10.48550/arXiv.2602.22625, fetched 2026-10-27 (unofficial)
