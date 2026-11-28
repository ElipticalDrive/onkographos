# Cluster Visibility Buffer Culling

Also called: Cluster Culling, Cluster Visibility Buffer, Visibility Buffer Culling, Visibility Culling via Visibility Buffer

## Overview
Cluster Visibility Buffer Culling is a technique to prune non-visible voxel clusters by storing visibility bits per cluster, avoiding full cell rasterization for occluded or back-facing groups [S1]. It differs from hierarchical Z-buffer approaches by using a visibility buffer texture updated via compute shaders and splatting, eliminating the need for pre-computed cluster bounds or normal cones [S1].

## How it works
**Inputs:**
- Active grid cells extracted from voxel data, grouped into clusters [S1].
- Cluster bounding boxes stored in the last metacell of each cluster [S1].
- Visibility buffer storing per-cluster visibility bits from the previous frame [S1].

**Outputs:**
- A list of visible clusters for full cell rasterization [S1].
- Updated visibility buffer indicating current frame's visible clusters [S1].

**Data Structures:**
- **Cluster Bound:** Stored in the last metacell of each cluster. It is an axis-aligned bounding box (AABB) [S1].
- **Visibility Buffer:** An `RG32UI` texture (two 32-bit unsigned integer channels) used to capture cluster IDs of visible cells during the cell rasterisation pass [S1].
- **ID Texture:** Generated during cell rasterization to capture cluster IDs of visible cells [S1].

**Steps:**
1. **Preparation:** During CPU-side isosurface extraction, divide volume data into blocks (e.g., $16 \times 16 \times 16$) and extract active grid cells in Morton order [S1]. Pad the last cluster of each block with empty metacells to ensure its size becomes 16, preventing spillover into different volume blocks [S1]. Store the cluster bound (AABB) in the last metacell of each cluster instead of cell metadata [S1]. Upload extracted metacells and grid cells to GPU buffer textures [S1].
2. **Initial Rasterization:** Perform cell rasterization for all clusters. During this pass, render cells as billboards [S1]. For each fragment rasterized for the billboard, perform a ray-box intersection test with the cell [S1].
3. **Capture Visibility:** Use an `RG32UI` texture as a visibility buffer to capture cluster IDs of visible cells during the cell rasterization pass [S1].
4. **Clear and Update:** After cell rasterization, clear the current visibility buffer bits [S1]. Update the visibility buffer from the ID texture in a separate compute shader pass [S1].
5. **Splatting for Non-Visible Clusters:** Render cells of non-visible clusters as single pixel splats to update the visibility buffer, similar to disk-based deferred splatting [S1]. This step identifies clusters that have become visible since the last frame [S1].
6. **Second Pass for New Visibility:** Introduce a second visibility buffer to record new clusters becoming visible [S1]. Perform full cell rasterization for these newly visible clusters in a second pass [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cluster_visibility_culling(clusters, prev_visibility_buffer, camera):
    # Step 1 & 2: Initial cell rasterization pass
    # Render cells as billboards. For each fragment, perform ray-box intersection with the cell.
    # Capture cluster IDs of visible cells into an ID texture.
    id_texture = rasterize_cells_billboards(clusters, camera)

    # Step 3 & 4: Clear and update visibility buffer
    # Clear the current visibility buffer bits.
    clear_texture(prev_visibility_buffer)
    # Update the visibility buffer from the ID texture in a separate compute shader pass.
    update_visibility_buffer_from_id(id_texture, prev_visibility_buffer)

    # Step 5: Splatting for non-visible clusters
    # Identify clusters not present in the updated visibility buffer.
    non_visible_clusters = [c for c in clusters if not prev_visibility_buffer.test_bit(c.id)]
    # Render cells of non-visible clusters as single pixel splats to update the visibility buffer.
    new_visibility_buffer = render_splats_non_visible(non_visible_clusters, camera)

    # Step 6: Second pass for newly visible clusters
    # Introduce a second visibility buffer to record new clusters becoming visible.
    # Perform full cell rasterisation for those clusters in a second pass.
    newly_visible_clusters = [c for c in clusters if new_visibility_buffer.test_bit(c.id) and not prev_visibility_buffer.test_bit(c.id)]
    rasterize_cells_billboards(newly_visible_clusters, camera)

    return prev_visibility_buffer

def rasterize_cells_billboards(clusters, camera):
    id_texture = create_empty_texture()
    for cluster in clusters:
        for cell in cluster.cells:
            for fragment in rasterize_billboard(cell, camera):
                if ray_box_intersection(cell, fragment.ray):
                    id_texture.write(fragment.coord, cluster.id)
    return id_texture

def clear_texture(texture):
    texture.fill(0)

def update_visibility_buffer_from_id(id_texture, visibility_buffer):
    for coord in id_texture.coords:
        cluster_id = id_texture.read(coord)
        if cluster_id != 0:
            visibility_buffer.atomic_or(cluster_id, 1)

def render_splats_non_visible(clusters, camera):
    new_buffer = create_empty_texture()
    for cluster in clusters:
        splat_coord = project_to_screen(cluster.center, camera)
        if not is_occluded(splat_coord, camera):
            new_buffer.atomic_or(cluster.id, 1)
    return new_buffer

def create_empty_texture():
    return Texture(width=0, height=0, channels=2, dtype=uint32)

def rasterize_billboard(cell, camera):
    # Standard rasterization of a screen-aligned quad representing the cell
    return [Fragment(coord=vec2(x, y), ray=ray(origin=cell.center, direction=normalize(camera.pos - cell.center))) for x in range(cell.screen_min_x, cell.screen_max_x) for y in range(cell.screen_min_y, cell.screen_max_y)]

def ray_box_intersection(cell, ray):
    # Standard AABB-ray intersection test
    return True  # Placeholder for standard math operation

def project_to_screen(center, camera):
    # Standard perspective projection
    return vec2(center.x * camera.proj[0][0], center.y * camera.proj[1][1])

def is_occluded(coord, camera):
    # Standard depth buffer test
    return False  # Placeholder for standard graphics operation
```

**Parameters:**
- **Cluster Size:** Up to 120 active grid cells per cluster [S1].
- **Block Size:** $16 \times 16 \times 16$ voxels per block [S1].
- **Metacells per Cluster:** 16 metacells per cluster segment [S1].

**Edge Cases and Pitfalls:**
- **Flickering Artifacts:** During fast camera movements, non-visible clusters becoming visible may cause flickering artifacts [S1]. This is mitigated by using a second visibility buffer to record new clusters becoming visible and performing full cell rasterization for them in a second pass [S1]. However, some flickering may still occur [S1].
- **Back-Face Culling:** Back-face culling is performed using normals computed in the tessellation shader from corner values [S1].
- **Conservative Culling:** The visibility culling is not conservative but avoids generating a hierarchical Z-buffer or storing cluster bounds/normal cones [S1].

## Subtopics

### Visibility Buffer Generation for Voxel Clusters
A technique to generate a depth and visibility buffer for voxel clusters by rasterizing cluster bounding boxes and ray tracing the internal $8^3$ grid to determine voxel occupancy, storing results in a 64-bit integer format [S2, unofficial]. It rasterizes the bounding box (rotated, translated cube) of each cluster using instancing and a simple vertex/fragment shader setup [S2, unofficial]. In the vertex shader, each cluster is transformed using its relative index inside the respective object and the object's world transform [S2, unofficial]. In the fragment shader, the $8^3$ grid of the cluster is ray-traced using the Amanatides et al. (1987) voxel traversal algorithm [S2, unofficial]. If a hit is detected, a 64-bit unsigned integer is composed containing: 24 bits for depth, 31 bits for the cluster index, and 9 bits for the voxel index [S2, unofficial]. The 64-bit unsigned integer stored in the visibility buffer has the following bit layout: bits 0-23 (24 bits) for depth, bits 24-54 (31 bits) for cluster index, bits 55-63 (9 bits) for voxel index [S2, unofficial]. Cluster grid size is $8^3$ voxels per cluster [S2, unofficial]. Voxel index bit width is 9 bits, allowing indexing of up to 512 voxels ($8^3$) [S2, unofficial]. Cluster index bit width is 31 bits, allowing indexing of over 2 billion clusters [S2, unofficial]. Depth bit width is 24 bits [S2, unofficial].

### Viewpoint-Based Visibility Filtering
A post-processing occlusion culling technique that clusters input cameras and pre-computes visibility masks for each cluster to accelerate test-time rendering [S3]. Input camera locations are clustered using k-means to obtain $k$ cluster centers, and each input camera is assigned to its nearest cluster center [S3]. Robustness is enhanced by adding random camera samples to the set of images used for calculating masks, ensuring coverage of potential test views [S3]. For rendering, a test viewpoint's camera center is assigned to the nearest cluster center, the pre-computed indicator list for that cluster is selected, and rasterization is performed only considering points marked as active in that list [S3]. The number of clusters $k$ is set to 64 for all scenes [S3]. The cluster threshold $t_{\text{cluster}}$ is set to 0.001 for all scenes [S3].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales with the number of clusters and cells. Performance may degrade with fast camera movements due to flickering artifacts [S1].
- **Strengths:** Avoids hierarchical Z-buffer generation and storage of cluster bounds/normal cones [S1]. Efficiently prunes non-visible clusters [S1].
- **Limitations:** Not conservative culling [S1]. Potential flickering artifacts during fast camera movements [S1].
- **When to Use:** Suitable for real-time isosurface rendering where performance is critical and conservative culling is not required [S1].
- **When Not to Use:** Not suitable when conservative culling is necessary or when flickering artifacts are unacceptable [S1].

## Background
Originated from the paper "Clustered Grid Cell Data Structure for Isosurface Rendering" [S1]. The method builds on deferred splatting techniques [S1]. It is used in real-time isosurface rendering and hybrid ray tracing applications [S1].

## Sources
[S1] Clustered Grid Cell Data Structure for Isosurface Rendering, https://doi.org/10.24132/jwscg.2020.28.2, fetched 2026-10-26
[S2] torben-gross/tg, https://github.com/torben-gross/tg, fetched 2026-10-26 (unofficial)
[S3] RadSplat: Radiance Field-Informed Gaussian Splatting for Robust Real- Time Rendering with 900+ FPS, https://doi.org/10.1109/3dv66043.2025.00018, fetched 2026-10-26
