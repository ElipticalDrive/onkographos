# Chunk Trees

Also called: replace span columns

## Overview
Chunk Trees are a planet-scale spatial partitioning structure for voxel terrain rendering, designed to replace the 'span columns' approach to eliminate data fragmentation and prevent visible refinement artifacts from ground level to orbit [S2, unofficial]. Unlike sibling methods that may rely on fixed grids or simpler octrees, Chunk Trees organize data into a planet-wide octree where each node contains 64 sub-trees of $4^3$ leaves, stored in shared pages to ensure uniform access and no fragmentation [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A voxel terrain dataset (splat data or generated fields) and a camera position/orientation [S2, unofficial].
*   **Output:** A rasterized image of the terrain, with rays marching restricted to individual chunks [S2, unofficial].

**Data Structures:**
The core structure is a planet-wide octree [S2, unofficial].
*   **Node Layout:** Each octree node contains 64 trees [S2, unofficial].
*   **Tree Layout:** Each tree has $4^3 = 64$ leaves [S2, unofficial].
*   **Storage:** Leaves are stored in shared pages accessible by any chunk. These pages contain surface and subsurface materials [S2, unofficial].
*   **Coordinate System:** The frame's origin snaps to 1,024 m cells to ensure motion vectors remain correct when the origin moves [S2, unofficial].

**Algorithm Steps:**

1.  **Construction:**
    *   Construct the chunk tree for the splat data, starting at index 0 [S1, unofficial].
    *   Use a log callback for diagnostics during construction [S1, unofficial].
    *   For generated terrain, evaluate a two-rate field (continent- and range-wide octaves) at lattice nodes and blend them [S2, unofficial].

2.  **Generation (GPU):**
    *   Evaluate the chunk tree on the GPU, once per chunk, within the frame's time budget [S2, unofficial].
    *   The budget adapts to both GPU and CPU availability, growing into remaining frame time [S2, unofficial].
    *   Cells that cannot be carved by caves are skipped to save computation [S2, unofficial].
    *   Blasts (destructive edits) regenerate within the frame's budget [S2, unofficial].

3.  **Residency Management:**
    *   Maintain a screen-space cut of the octree that follows the eye position [S2, unofficial].
    *   The drawn cut covers the whole world to prevent "checker-hole" artifacts [S2, unofficial].
    *   Tiles out of view are rendered at a coarser level [S2, unofficial].
    *   If a camera drops from orbit, refinement occurs over a few frames [S2, unofficial].
    *   Tiles still refining render as ground, not as boxes [S2, unofficial].

4.  **Motion-Aware Detail:**
    *   Calculate the eye's motion sweep in "flow pixels" per frame [S2, unofficial].
    *   Determine cell coverage: cells may cover $1 + \frac{\text{flow}}{16}$ pixels, up to a maximum of 3 pixels [S2, unofficial].
    *   The view center, a still camera, and turns maintain a 1-pixel coverage [S2, unofficial].

5.  **Rendering:**
    *   Rasterize and cull chunk boxes on the GPU into the seed's draw list [S2, unofficial].
    *   Restrict ray marching to inside one chunk [S2, unofficial].
    *   A chunk's box uses the slope of its surface [S2, unofficial].
    *   Levels crossfade into each other [S2, unofficial].
    *   Rays continue across cube face edges [S2, unofficial].
    *   From level 13 onwards, rays walk the grid's own cells (planes and spheres about the centre) to ensure tiles and faces are watertight from orbit [S2, unofficial].

6.  **Edits (Optional/Destructible Terrain):**
    *   Edits apply in layers: recent brushes exact, older small ones sealed into bricks, large ones analytic [S2, unofficial].
    *   An edit tree keeps the surfaces edits leave, not their whole history [S2, unofficial].
    *   Saves store layers bit-exact, keeping memory bounded [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_chunk_tree(camera, frame_budget_ms, octree):
    # 1. Frame origin snapping
    origin = snap_to_grid(camera.position, 1024.0)
    camera.origin = origin

    # 2. Residency: screen-space cut following eye
    active_chunks = get_screen_space_cut(camera, octree)

    # 3. Generation phase (GPU)
    start_time = get_gpu_time()
    for chunk in active_chunks:
        if not can_carve_by_caves(chunk):
            continue
        generate_chunk_gpu(chunk)
        if get_gpu_time() - start_time > frame_budget_ms:
            break

    # 4. Motion-aware detail
    flow_pixels = calculate_flow_pixels(camera.prev_position, camera.position)
    for chunk in active_chunks:
        coverage = min(3.0, 1.0 + flow_pixels / 16.0)
        if camera.is_still_or_turning():
            coverage = 1.0
        chunk.set_detail_level(coverage)

    # 5. Rendering phase
    draw_list = []
    for chunk in active_chunks:
        if is_in_view(chunk, camera):
            box = rasterize_chunk_box(chunk)
            draw_list.append(box)

    # 6. Ray marching
    for ray in generate_rays_from_camera(camera):
        chunk = get_chunk_containing(ray.origin, octree)
        if chunk is None:
            continue
        hit = march_ray_in_chunk(ray, chunk)
        if chunk.level >= 13:
            hit = watertight_march(ray, chunk)
        if hit:
            set_pixel(ray.screen_pos, compute_color(hit))

def snap_to_grid(position, cell_size):
    return [int(p // cell_size) * cell_size for p in position]

def get_screen_space_cut(camera, octree):
    # Returns chunks within the camera's frustum and refinement radius
    return octree.query_frustum(camera.frustum)

def get_gpu_time():
    pass  # Not given in the sources.

def can_carve_by_caves(chunk):
    return chunk.has_cave_carvable_cells

def generate_chunk_gpu(chunk):
    pass  # Not given in the sources.

def calculate_flow_pixels(prev_pos, curr_pos):
    pass  # Not given in the sources.

def is_in_view(chunk, camera):
    return camera.frustum.intersects(chunk.bounding_box)

def rasterize_chunk_box(chunk):
    return chunk.bounding_box

def get_chunk_containing(point, octree):
    return octree.find_node_containing(point)

def march_ray_in_chunk(ray, chunk):
    pass  # Not given in the sources.

def watertight_march(ray, chunk):
    pass  # Not given in the sources.

def set_pixel(screen_pos, color):
    pass

def compute_color(hit):
    return (0.0, 0.0, 0.0)
```

**Parameters:**
*   **Chunk Leaf Size:** $4^3$ voxels per tree [S2, unofficial].
*   **Trees per Node:** 64 [S2, unofficial].
*   **Grid Snapping:** 1,024 m [S2, unofficial].
*   **Motion Detail Max Coverage:** 3 pixels [S2, unofficial].
*   **Watertight Threshold:** Level 13 [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Checker-Hole Artifacts:** Prevented by ensuring the drawn cut covers the whole world [S2, unofficial].
*   **Refinement Artifacts:** Eliminated by the chunk tree structure, replacing span columns [S2, unofficial].
*   **Register Regression:** A walk's register regression was identified and removed to improve performance [S2, unofficial].
*   **Host Shared Device:** The pass never polls the host's shared device [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Generation is bounded by the frame budget, adapting to GPU and CPU availability [S2, unofficial].
*   **Memory Complexity:** Memory is bounded by storing layers bit-exact and using shared pages for leaves, avoiding fragmentation [S2, unofficial].
*   **Measured Performance (RTX 3060):**
    *   **1080p:** Ground primary 1.48 ms, sun and sky 2.06 ms; Mountain primary 3.10 ms, sun and sky 2.78 ms. Mountain view terrain total: 6.60 ms [S2, unofficial].
    *   **1440p:** Terrain costs dropped from 9.95 ms to 5.85 ms after removing a walk's register regression [S2, unofficial].
*   **Strengths:** Eliminates fragmentation, prevents refinement artifacts, supports planet-scale destructible terrain, and adapts to hardware budgets [S2, unofficial].
*   **Limitations:** Complex implementation due to the nested octree/tree structure and motion-aware detail calculations [S2, unofficial].

## Background
Chunk Trees were introduced in the Helio project to replace the 'span columns' approach for voxel terrain rendering [S2, unofficial]. The design aims to provide a fully destructible planet with efficient rendering from ground to orbit [S2, unofficial]. The method was documented in the context of the Pulsar engine and the Helio voxel pass [S2, unofficial].

## Sources
[S1] sparkjsdev/spark, https://github.com/sparkjsdev/spark, fetched 2026-10-27 (unofficial)
[S2] Voxel planet: chunk trees, a fully destructible planet, and the lighting to show it, https://github.com/Far-Beyond-Pulsar/Helio/pull/327, fetched 2026-10-28 (unofficial)
