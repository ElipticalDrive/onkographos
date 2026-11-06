# Voxel Splatting to Pixel Footprints

Also called: voxel splat

## Overview
A rasterization technique where each voxel in a cluster is processed by a single invocation, projecting its footprint box to the pixel centers it covers, replacing the previous method of walking every pixel of a brick's rectangle through its cells [S1, unofficial]. It solves the inefficiency of processing pixels that no voxel footprint covers, particularly at shallow voxel levels where footprints are small fractions of cell sizes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A cluster of voxels, their associated brick metadata (first voxel indices, masks), and the camera/projection matrices.
*   **Output:** A visibility buffer or hit list containing the nearest voxel for each covered pixel, determined via atomic operations.

**Data Structures:**
*   **Brick Metadata:** Contains an array of first voxel indices for binary search and a bitmask indicating which cells within the brick are occupied.
*   **Voxel Footprint Box:** Defined as a capsule widened by its radius, constrained within its cell boundaries. The box's corners lie on a grid of 32nds of a cell.

**Steps:**
1.  **Invocation:** Invoke one shader invocation per voxel of the cluster [S1, unofficial].
2.  **Brick Identification:** Determine the brick containing the voxel by performing a binary search over the bricks' first voxels [S1, unofficial].
3.  **Cell Index:** Determine the cell index within the brick by finding the $n$-th set bit of the brick's mask [S1, unofficial].
4.  **Footprint Definition:** Define the voxel's footprint box as a capsule widened by its radius, constrained within its cell [S1, unofficial].
5.  **Projection Calculation:** Project the footprint box to clip space. The center and half-extents are reached as steps along the brick's grid of 32nds of a cell. The projection lies within $(r_x + |x_c / w_c| r_w) / (w_c - r_w)$ of the center in x, and likewise in y, widened by a 64th of a pixel [S1, unofficial].
    *   Here, $r_x$ is the x-radius, $x_c$ is the center x, $w_c$ is the center w, and $r_w$ is the w-radius.
6.  **Edge Case Handling:**
    *   If a voxel is wholly behind the near plane, it covers no pixels [S1, unofficial].
    *   If a voxel reaches behind the camera's plane, it covers the whole target [S1, unofficial].
7.  **Pair Pooling:** A subgroup pools its voxels' (voxel, pixel) pairs using an exclusive add, followed by a binary search per pair [S1, unofficial].
8.  **Ray Testing:** Fill the shader lanes with rays' tests for whichever voxels cover pixels [S1, unofficial].
9.  **Hit Test:** For each pair, test the ray against that voxel alone using `vis_voxel_hit` [S1, unofficial].
    *   Inside `vis_voxel_hit`: Take the ray's span through the voxel's cell from the cell's own planes, then compute the footprint over that span [S1, unofficial].
10. **Atomic Update:** On a hit, perform an atomic max of $(1 / t, \text{cluster}, \text{brick})$ so that the nearest voxel wins [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for vis_voxel_raster.comp
# One invocation per voxel in the cluster

def process_voxel(voxel_id, cluster_data):
    # 1. Identify Brick
    brick_idx = binary_search_bricks(cluster_data.bricks, cluster_data.first_voxel_indices, voxel_id)
    brick = cluster_data.bricks[brick_idx]
    
    # 2. Identify Cell
    cell_idx = find_nth_set_bit(brick.mask, voxel_id - brick.first_voxel_index)
    
    # 3. Define Footprint Box
    # Capsule widened by radius, constrained within cell
    footprint_box = define_footprint_box(voxel, brick.cell_size)
    
    # 4. Project to Clip Space
    # Corners on grid of 32nds of a cell
    center_clip = world_to_clip(footprint_box.center)
    half_extents_clip = world_to_clip(footprint_box.half_extents)
    
    # Calculate projection bounds
    # widened by 64th of a pixel
    rx = footprint_box.radius_x
    xc = center_clip.x
    wc = center_clip.w
    rw = footprint_box.radius_w
    
    x_proj_bound = (rx + abs(xc / wc) * rw) / (wc - rw)
    y_proj_bound = (footprint_box.radius_y + abs(center_clip.y / wc) * footprint_box.radius_w) / (wc - footprint_box.radius_w)
    
    # 5. Edge Cases
    if center_clip.z > 0: # Wholly behind near plane (assuming NDC z>0 is behind)
        return
    
    if wc < 0: # Reaching behind camera plane
        # Covers whole target
        target_pixels = get_all_target_pixels()
    else:
        # Determine covered pixel centers
        target_pixels = get_pixel_centers_covered_by_box(center_clip, x_proj_bound, y_proj_bound)
        
    # 6. Pool Pairs
    # Subgroup exclusive add to count pairs, then binary search to assign lanes
    pair_list = [(voxel_id, pixel) for pixel in target_pixels]
    pooled_pairs = subgroup_pool(pair_list)
    
    # 7. Ray Tests
    for voxel_id, pixel in pooled_pairs:
        ray = get_ray_for_pixel(pixel)
        
        # 8. Hit Test
        hit_t = vis_voxel_hit(ray, voxel, brick)
        
        if hit_t is not None:
            # 9. Atomic Max
            # Nearest voxel wins (smallest t, largest 1/t)
            atomic_max_hit(1.0 / hit_t, cluster_id, brick_id)

def vis_voxel_hit(ray, voxel, brick):
    # Take ray's span through voxel's cell from cell's own planes
    t_min, t_max = cell_plane_intersection(ray, brick.cell_planes)
    
    # Compute footprint over that span
    if footprint_intersects_span(voxel.footprint, t_min, t_max):
        return t_min
    return None
```

## Operating characteristics
*   **Time Complexity:** Proportional to the number of voxels in the cluster, rather than the area of the brick's bounding rectangle. This is significantly faster when voxel footprints are small relative to cell sizes (e.g., at shallow levels) [S1, unofficial].
*   **Memory Complexity:** Requires storing brick metadata (first voxel indices, masks) and temporary buffers for subgroup pooling [S1, unofficial].
*   **Performance:** In tests on `foliage_4k_production`, the previous method sent 14.9 M rays per frame, taking 59.7 M traversal steps and 15.2 M footprint tests for 1.78 M hits. The splatting method reduces this by only testing voxels that actually cover pixel centers [S1, unofficial].
*   **Strengths:** Eliminates wasted computation on pixels not covered by any voxel footprint. Efficient for sparse data where footprints are small.
*   **Limitations:** Requires subgroup operations for pooling, which may have overhead on hardware with poor subgroup support. Precision issues may arise in projection calculations near the camera plane.

## Background
This method was introduced as part of stage 3 rasterization updates in the Nepheline renderer, specifically in ADR-0061 [S1, unofficial]. It replaced the DDA-based traversal (`vis_voxel_trace`) and the previous pixel-walking brick rasterization, aiming to optimize performance for scenes with foliage and other complex geometry where voxel footprints are small [S1, unofficial].

## Sources
[S1] feat(render): voxels splatted to the pixels their footprints cover (ADR-0061), https://github.com/SugalAbubaker/nepheline/pull/115, fetched 2026-10-27 (unofficial)
