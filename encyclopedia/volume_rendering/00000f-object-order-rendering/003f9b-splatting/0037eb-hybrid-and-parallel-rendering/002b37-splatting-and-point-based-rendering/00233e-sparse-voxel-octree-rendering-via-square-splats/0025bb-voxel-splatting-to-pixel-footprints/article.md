# Voxel Splatting to Pixel Footprints

Also called: voxel splat

## Overview
A rasterization technique where each voxel in a cluster is processed by a single invocation, projecting its footprint box to the pixel centers it covers, replacing the previous method of walking every pixel of a brick's rectangle through its cells [S1, unofficial]. It solves the inefficiency of processing pixels that no voxel footprint covers, particularly at shallow voxel levels where footprints are small fractions of cell sizes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A cluster of voxels, their associated brick metadata (first voxel indices, masks), and the camera/projection matrices [S1, unofficial].
*   **Output:** A visibility buffer or hit list containing the nearest voxel for each covered pixel, determined via atomic operations [S1, unofficial].

**Data Structures:**
*   **Brick Metadata:** Contains an array of first voxel indices for binary search and a bitmask indicating which cells within the brick are occupied [S1, unofficial].
*   **Voxel Footprint Box:** Defined as a capsule widened by its radius, constrained within its cell boundaries [S1, unofficial]. The box's corners lie on a grid of 32nds of a cell [S1, unofficial].

**Steps:**
1.  **Invocation:** Invoke one shader invocation per voxel of the cluster [S1, unofficial].
2.  **Brick Identification:** Determine the brick containing the voxel by performing a binary search over the bricks' first voxels [S1, unofficial].
3.  **Cell Index:** Determine the cell index within the brick by finding the $n$-th set bit of the brick's mask [S1, unofficial].
4.  **Footprint Definition:** Define the voxel's footprint box as a capsule widened by its radius, constrained within its cell [S1, unofficial].
5.  **Projection Calculation:** Project the footprint box to clip space [S1, unofficial]. The center and half-extents are reached as steps along the brick's grid of 32nds of a cell [S1, unofficial]. The projection lies within $(r_x + |x_c / w_c| r_w) / (w_c - r_w)$ of the center in x, and likewise in y, widened by a 64th of a pixel [S1, unofficial]. Here, $r_x$ is the x-radius, $x_c$ is the center x, $w_c$ is the center w, and $r_w$ is the w-radius [S1, unofficial].
6.  **Edge Case Handling:** If a voxel is wholly behind the near plane, it covers no pixels [S1, unofficial]. If a voxel reaches behind the camera's plane, it covers the whole target [S1, unofficial].
7.  **Pair Pooling:** A subgroup pools its voxels' (voxel, pixel) pairs using an exclusive add, followed by a binary search per pair [S1, unofficial].
8.  **Ray Testing:** Fill the shader lanes with rays' tests for whichever voxels cover pixels [S1, unofficial].
9.  **Hit Test:** For each pair, test the ray against that voxel alone using `vis_voxel_hit` [S1, unofficial]. Inside `vis_voxel_hit`, take the ray's span through the voxel's cell from the cell's own planes, then compute the footprint over that span [S1, unofficial].
10. **Atomic Update:** On a hit, perform an atomic max of $(1 / t, \text{cluster}, \text{brick})$ so that the nearest voxel wins [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for vis_voxel_raster.comp
# One invocation per voxel in the cluster

def process_voxel(voxel_id, cluster_data, mvp_matrix):
    # 1. Identify Brick (binary search over first voxel indices) [G]
    first_voxels = cluster_data.first_voxel_indices
    left, right = 0, len(first_voxels) - 1
    brick_idx = right
    while left <= right:
        mid = (left + right) // 2
        if first_voxels[mid] <= voxel_id:
            brick_idx = mid
            left = mid + 1
        else:
            right = mid - 1
    brick = cluster_data.bricks[brick_idx]

    # 2. Identify Cell (find n-th set bit in mask) [G]
    offset = voxel_id - brick.first_voxel_index
    cell_idx = 0
    mask = brick.mask
    for i in range(32):
        if (mask >> i) & 1:
            if offset == cell_idx:
                break
            cell_idx += 1

    # 3. Define Footprint Box
    # Capsule widened by radius, constrained within cell
    cell_size = brick.cell_size
    voxel_center = voxel_data.center
    radius = voxel_data.radius
    min_corner = max(voxel_center - radius, brick.min_corner)
    max_corner = min(voxel_center + radius, brick.max_corner)
    footprint_center = (min_corner + max_corner) / 2.0
    footprint_half_extents = (max_corner - min_corner) / 2.0

    # 4. Project to Clip Space
    # Corners lie on grid of 32nds of a cell; center/half-extents reached as steps along grid [S1, unofficial]
    clip_center = multiply_mvp(mvp_matrix, [footprint_center.x, footprint_center.y, footprint_center.z, 1.0])
    clip_half_ext = multiply_mvp(mvp_matrix, [footprint_half_extents.x, footprint_half_extents.y, footprint_half_extents.z, 0.0])

    rx = footprint_half_extents.x
    xc = clip_center[0]
    wc = clip_center[3]
    rw = clip_half_ext[3]

    # Projection bounds, widened by 64th of a pixel [S1, unofficial]
    x_proj_bound = (rx + abs(xc / wc) * rw) / (wc - rw) + 1.0 / 64.0
    y_proj_bound = (footprint_half_extents.y + abs(clip_center[1] / wc) * rw) / (wc - rw) + 1.0 / 64.0

    # 5. Edge Cases
    # Wholly behind near plane covers nothing [S1, unofficial]
    if clip_center[2] > 0:
        return
    # Reaching behind camera plane takes whole target [S1, unofficial]
    if wc < 0:
        target_pixels = list(range(cluster_data.resolution_x * cluster_data.resolution_y))
    else:
        # Determine covered pixel centers from clip-space bounds [G]
        target_pixels = compute_pixel_centers_from_clip_bounds(clip_center, x_proj_bound, y_proj_bound, cluster_data.resolution_x, cluster_data.resolution_y)

    # 6. Pool Pairs
    # Subgroup exclusive add, then binary search per pair [S1, unofficial]
    pair_list = [(voxel_id, pixel) for pixel in target_pixels]
    pooled_pairs = subgroup_pool_pairs(pair_list)

    # 7. Ray Tests
    for vid, pixel in pooled_pairs:
        ray = generate_ray_for_pixel(pixel, cluster_data.camera)

        # 8. Hit Test
        hit_t = vis_voxel_hit(ray, voxel_data, brick)

        if hit_t is not None:
            # 9. Atomic Max
            # Nearest voxel wins (smallest t, largest 1/t) [S1, unofficial]
            atomic_max_hit(1.0 / hit_t, cluster_data.id, brick.id)

def vis_voxel_hit(ray, voxel, brick):
    # Take ray's span through voxel's cell from the cell's own planes [S1, unofficial]
    t_min, t_max = ray_aabb_intersection(ray, brick.cell_planes)
    if t_min is None or t_max is None or t_max < 0:
        return None

    # Compute the footprint over that span [S1, unofficial]
    if footprint_overlaps_span(voxel.footprint, t_min, t_max):
        return t_min
    return None

def ray_aabb_intersection(ray, planes):
    # Iterative slab method [G]
    t_near = -float('inf')
    t_far = float('inf')
    for i in range(3):
        if abs(ray.dir[i]) < 1e-8:
            if ray.origin[i] < planes[i].min or ray.origin[i] > planes[i].max:
                return None, None
        else:
            t1 = (planes[i].min - ray.origin[i]) / ray.dir[i]
            t2 = (planes[i].max - ray.origin[i]) / ray.dir[i]
            t_near = max(t_near, min(t1, t2))
            t_far = min(t_far, max(t1, t2))
    if t_near > t_far:
        return None, None
    return t_near, t_far

def footprint_overlaps_span(footprint, t_min, t_max):
    # Check if footprint intersects ray segment [G]
    return t_min <= footprint.max_t and t_max >= footprint.min_t

def multiply_mvp(matrix, vec):
    # Standard 4x4 matrix-vector multiplication [G]
    out = [0.0, 0.0, 0.0, 0.0]
    for r in range(4):
        for c in range(4):
            out[r] += matrix[r][c] * vec[c]
    return [out[0]/out[3], out[1]/out[3], out[2]/out[3], out[3]]

def compute_pixel_centers_from_clip_bounds(center_clip, x_bound, y_bound, res_x, res_y):
    # Map clip-space bounds to pixel indices [G]
    pixels = []
    for y in range(int(center_clip[1] - y_bound), int(center_clip[1] + y_bound) + 1):
        for x in range(int(center_clip[0] - x_bound), int(center_clip[0] + x_bound) + 1):
            if 0 <= x < res_x and 0 <= y < res_y:
                pixels.append(y * res_x + x)
    return pixels

def subgroup_pool_pairs(pair_list):
    # Exclusive add to count pairs, then binary search per pair to assign lanes [G]
    # Simplified representation of hardware subgroup exclusive scan + binary search assignment
    counts = [0] * len(pair_list)
    for i in range(1, len(pair_list)):
        counts[i] = counts[i-1] + 1
    assigned = []
    for i, (vid, pix) in enumerate(pair_list):
        lane_idx = counts[i]
        assigned.append((vid, pix, lane_idx))
    return assigned
```

## Operating characteristics
*   **Time Complexity:** Proportional to the number of voxels in the cluster, rather than the area of the brick's bounding rectangle [S1, unofficial].
*   **Memory Complexity:** Not given in the sources [S1, unofficial].
*   **Performance:** In tests on `foliage_4k_production`, the previous method sent 14.9 M rays per frame, taking 59.7 M traversal steps and 15.2 M footprint tests for 1.78 M hits [S1, unofficial].
*   **Strengths:** Eliminates wasted computation on pixels not covered by any voxel footprint [S1, unofficial].
*   **Limitations:** Requires subgroup operations for pooling, which may have overhead on hardware with poor subgroup support [S1, unofficial].

## Background
This method was introduced as part of stage 3 rasterization updates in the Nepheline renderer, specifically in ADR-0061 [S1, unofficial]. It replaced the DDA-based traversal (`vis_voxel_trace`) and the previous pixel-walking brick rasterization, aiming to optimize performance for scenes with foliage and other complex geometry where voxel footprints are small [S1, unofficial].

## Sources
[S1] feat(render): voxels splatted to the pixels their footprints cover (ADR-0061), https://github.com/SugalAbubaker/nepheline/pull/115, fetched 2026-10-27 (unofficial)
