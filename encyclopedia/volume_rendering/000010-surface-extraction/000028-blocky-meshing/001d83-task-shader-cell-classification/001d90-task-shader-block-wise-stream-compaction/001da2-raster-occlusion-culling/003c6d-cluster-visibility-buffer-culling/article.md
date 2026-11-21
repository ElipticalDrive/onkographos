# Cluster Visibility Buffer Culling

Also called: Cluster Culling, Cluster Visibility Buffer, Visibility Buffer Culling, Visibility Culling via Visibility Buffer

## Overview
Cluster Visibility Buffer Culling is a technique to prune non-visible voxel clusters by storing visibility bits per cluster, avoiding full cell rasterization for occluded or back-facing groups [S1]. It differs from hierarchical Z-buffer approaches by using a visibility buffer texture updated via compute shaders and splatting, eliminating the need for pre-computed cluster bounds or normal cones [S1].

## How it works
**Inputs:**
- Active grid cells extracted from voxel data, grouped into clusters [S1].
- Cluster bounding boxes stored in the last metacell of each cluster [S1].
- Previous frame's visibility buffer state [S1].

**Outputs:**
- A list of visible clusters for full cell rasterization [S1].
- Updated visibility buffer indicating current frame's visible clusters [S1].

**Data Structures:**
- **Cluster Bound:** Stored in the last metacell of each cluster. It is an axis-aligned bounding box (AABB) [S1].
- **Visibility Buffer:** An `RG32UI` texture (two 32-bit unsigned integer channels) [S1]. One channel typically stores the visibility bit or cluster ID, and the other may store auxiliary data or be unused depending on implementation specifics [S1].
- **ID Texture:** Generated during cell rasterization to capture cluster IDs of visible cells [S1].

**Steps:**
1. **Preparation:** During CPU-side isosurface extraction, divide volume data into blocks (e.g., $16 \times 16 \times 16$) and extract active grid cells in Morton order [S1]. Pad the last cluster of each block with empty metacells to ensure its size becomes 16, preventing spillover into different volume blocks [S1]. Store the cluster bound (AABB) in the last metacell of each cluster instead of cell metadata [S1]. Upload extracted metacells and grid cells to GPU buffer textures [S1].
2. **Initial Rasterization:** Perform cell rasterization for all clusters. During this pass, render cells as billboards/splats. For each fragment, perform a ray-box intersection test with the cell [S1].
3. **Capture Visibility:** Use an `RG32UI` texture as a visibility buffer to capture cluster IDs of visible cells during the cell rasterization pass [S1].
4. **Clear and Update:** After cell rasterization, clear the current visibility buffer bits [S1]. Update the visibility buffer from the ID texture in a separate compute shader pass [S1].
5. **Splatting for Non-Visible Clusters:** Render cells of non-visible clusters as single pixel splats to update the visibility buffer, similar to disk-based deferred splatting [S1]. This step identifies clusters that have become visible since the last frame [S1].
6. **Second Pass for New Visibility:** Introduce a second visibility buffer to record new clusters becoming visible [S1]. Perform full cell rasterization for these newly visible clusters in a second pass [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Cluster Visibility Buffer Culling
# Assumes GPU compute shader and rasterization pipeline access

def cluster_visibility_culling(clusters, previous_visibility_buffer, camera):
    """
    clusters: List of cluster objects, each containing:
        - cells: List of active grid cells
        - bound: AABB stored in last metacell
        - id: Unique cluster identifier
    previous_visibility_buffer: RG32UI texture from previous frame
    camera: Current camera state
    """
    
    # Step 1: Initial Rasterization Pass
    # Render all cells. Capture cluster IDs of visible cells into an ID texture.
    id_texture = render_cells(clusters, camera)
    
    # Step 2: Clear Visibility Buffer
    clear_texture(previous_visibility_buffer)
    
    # Step 3: Update Visibility Buffer from ID Texture
    # Use a compute shader to aggregate cluster IDs from the ID texture
    # into the visibility buffer.
    update_visibility_buffer_compute(id_texture, previous_visibility_buffer)
    
    # Step 4: Identify Non-Visible Clusters
    # Clusters not present in the updated visibility buffer are considered non-visible
    # for the current frame, except for those that might have just become visible.
    non_visible_clusters = get_clusters_not_in_buffer(clusters, previous_visibility_buffer)
    
    # Step 5: Splatting for Non-Visible Clusters
    # Render cells of non-visible clusters as single pixel splats
    # to detect if they have become visible.
    new_visibility_buffer = render_splats(non_visible_clusters, camera)
    
    # Step 6: Merge Visibility States
    # Combine previous and new visibility states
    final_visibility_buffer = merge_visibility_buffers(previous_visibility_buffer, new_visibility_buffer)
    
    # Step 7: Second Pass for Newly Visible Clusters
    # Identify clusters that are newly visible
    newly_visible_clusters = get_newly_visible_clusters(final_visibility_buffer, previous_visibility_buffer)
    
    # Perform full cell rasterization for newly visible clusters
    render_cells(newly_visible_clusters, camera)
    
    return final_visibility_buffer

def update_visibility_buffer_compute(id_texture, visibility_buffer):
    """
    Compute shader to update visibility buffer based on ID texture.
    """
    # For each pixel in id_texture:
    #   cluster_id = id_texture.read(pixel_coord).r
    #   if cluster_id != 0:
    #       visibility_buffer.atomic_or(cluster_id, 1)
    pass

def render_splats(clusters, camera):
    """
    Render clusters as single pixel splats.
    """
    # For each cluster in clusters:
    #   if is_backfacing(cluster, camera):
    #       continue
    #   if is_occluded(cluster, camera):
    #       continue
    #   render_single_pixel_splat(cluster, camera)
    pass
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

## Operating characteristics
- **Time Complexity:** Depends on the number of clusters and cells. Initial rasterization is $O(N_{cells})$, visibility buffer update is $O(N_{visible\_cells})$, and splatting is $O(N_{non\_visible\_clusters})$.
- **Memory Complexity:** $O(N_{clusters})$ for visibility buffer storage.
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
