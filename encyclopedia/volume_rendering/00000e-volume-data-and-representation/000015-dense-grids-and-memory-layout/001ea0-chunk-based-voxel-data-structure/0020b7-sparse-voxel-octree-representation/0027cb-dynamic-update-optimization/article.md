# Dynamic Update Optimization

Also called: Dynamic Voxel Cone Tracing Optimization

## Overview
Dynamic Update Optimization is a set of heuristics and data structure modifications applied to Voxel Cone Tracing to accelerate indirect illumination calculations in scenes containing both static and dynamic geometry [S1, unofficial]. It solves the performance bottleneck of re-voxelizing and re-computing lighting for the entire scene every frame by isolating dynamic updates to affected regions [S1, unofficial]. It sets itself apart from static-only voxelization by maintaining a hybrid state where static geometry is pre-computed and cached, while dynamic geometry is processed with relaxed accuracy constraints and targeted re-filtering [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   Scene geometry (static and dynamic meshes).
*   Camera position, orientation, field of view (FOV), and near/far planes.
*   Maximum cone tracing distance ($D_{max}$).
*   Voxel resolution (e.g., $256 \times 256 \times 256$).
*   Screen resolution ($W \times H$).

**Outputs:**
*   Updated Sparse Voxel Octree (SVo) containing static and dynamic voxel data.
*   Indirect lighting buffer (screen-space).

### Data Structures
The method modifies the standard SVo node structure to support dynamic updates:
1.  **Octree Node Buffer:** Stores voxel data. Dynamic voxel fragments are appended to the end of this buffer rather than overwriting existing static nodes.
2.  **Node Flags:** A bit flag stored in each node to identify if it is static or dynamic. Static nodes are never overwritten; dynamic nodes are appended.
3.  **Voxel Data Layout:**
    *   **Static Nodes:** Store full attributes (diffuse color, average normal, etc.).
    *   **Dynamic Nodes:** Store a 1-byte counter and the normal vector. Diffuse color is computed or approximated differently (see Step 2) [S1, unofficial].
4.  **Neighbor Offsets:** Pointers/indices linking nodes to their neighbors. Static nodes pointing to dynamic neighbors require reset/update when dynamic data changes.
5.  **Affected Node Buffer:** A queue used to store indices of static nodes that need re-filtering due to changes in adjacent dynamic voxels.

### Numbered Steps

1.  **Initialization (Static Geometry):**
    *   Perform the standard 4 steps of voxel cone tracing (voxelization, building octree, filtering, cone tracing) for all static geometry [S1, unofficial].
    *   Mark these nodes as static using the bit flag.
    *   Store them in the octree.

2.  **Per-Frame Dynamic Voxelization:**
    *   Voxelize only dynamic objects [S1, unofficial].
    *   For each dynamic fragment, update the octree node's normal and diffuse attributes using `InterlockedMax()` instead of atomic averaging [S1, unofficial].
        *   **Normal:** Use `InterlockedMax()` on the normal vector components. This is faster than averaging but reduces lighting quality for small voxel volumes (e.g., $256^3$), causing artifacts where light reflects incorrectly compared to average normals. To mitigate, `InterlockedMax()` is used *only* for dynamic geometry normals; static geometry retains average normal calculation [S1, unofficial].
        *   **Diffuse:** Use `InterlockedMax()` for diffuse color as well [S1, unofficial].
        *   **Counter:** Maintain a 1-byte counter in the voxel data to track how many fragments contributed to the node (necessary because `InterlockedMax` does not provide an average count) [S1, unofficial].
    *   Append dynamic voxel fragments to the end of the octree node buffer [S1, unofficial].

3.  **View-Frustum Culling for Direct Lighting Injection:**
    *   When injecting direct lighting into the octree, perform view-frustum culling to avoid filtering light far from the camera [S1, unofficial].
    *   Calculate an **Extended Frustum**:
        *   Move the camera backward slightly.
        *   Increase the far plane.
        *   Keep the same FOV.
        *   The extended distance is calculated as: $D_{ext} = D_{max} + \text{filtered voxel size}$ [S1, unofficial].
    *   Only voxels within this extended frustum are processed for direct lighting injection [S1, unofficial].

4.  **Half-Resolution Cone Tracing with Edge Correction:**
    *   Perform the initial cone tracing pass at half screen resolution ($W/2 \times H/2$) [S1, unofficial].
    *   **Artifact Handling:** Upscaling to full resolution causes visible artifacts, especially at geometry edges [S1, unofficial].
        *   **Edge Detection:** Use a depth buffer edge detection filter to identify pixels at geometry boundaries [S1, unofficial].
        *   **Re-tracing:** Perform cone tracing at full resolution *only* for these detected edge pixels [S1, unofficial].
        *   **Fallback:** If re-tracing is too expensive, blur the edge pixels by averaging with neighbors (lower quality, higher performance) [S1, unofficial].

5.  **Static Node Re-filtering:**
    *   Identify static octree nodes affected by dynamic voxels from the previous and current frames [S1, unofficial].
    *   Dispatch threads for all nodes to check for changes and queue affected node indices into the **Affected Node Buffer** [S1, unofficial].
    *   Reset static octree node neighbor offsets that point to dynamic nodes [S1, unofficial].
    *   Re-filter only the static nodes listed in the buffer [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dynamic_update_optimization(scene, camera, dt):
    # Step 1: Initialization (One-time or when static geometry changes)
    if scene.needs_initialization:
        static_geometry = scene.get_static_geometry()
        octree = build_octree(static_geometry)
        mark_nodes_static(octree)
        scene.needs_initialization = False

    # Step 2: Dynamic Voxelization
    dynamic_geometry = scene.get_dynamic_geometry()
    dynamic_fragments = voxelize(dynamic_geometry)
    
    for fragment in dynamic_fragments:
        node = octree.get_or_create_node(fragment.position)
        # Use InterlockedMax for speed, accepting quality loss for dynamic normals
        node.normal = interlocked_max(node.normal, fragment.normal)
        node.diffuse = interlocked_max(node.diffuse, fragment.color)
        node.counter += 1
        # Append to end of buffer if new
        if node.is_new:
            octree.append_to_dynamic_buffer(node)

    # Step 3: View-Frustum Culling for Direct Lighting
    extended_far = camera.far_plane + (max_cone_tracing_distance + voxel_size)
    extended_frustum = create_extended_frustum(camera, extended_far)
    
    direct_lighting_buffer = compute_direct_lighting(octree, extended_frustum)

    # Step 4: Cone Tracing (Half Resolution + Edge Correction)
    half_res_buffer = cone_trace(octree, camera, resolution=(screen_w/2, screen_h/2))
    
    # Detect edges using depth buffer
    edge_mask = detect_edges(camera.depth_buffer)
    
    # Re-trace edges at full resolution
    full_res_buffer = half_res_buffer.upsample(bilinear=True)
    edge_pixels = get_pixel_indices(edge_mask)
    
    for idx in edge_pixels:
        full_res_buffer[idx] = cone_trace_pixel(octree, camera, idx, resolution=(screen_w, screen_h))

    # Step 5: Static Node Re-filtering
    affected_nodes = find_affected_static_nodes(octree, dynamic_fragments)
    
    for node in affected_nodes:
        reset_node_neighbor_offsets(node)
        re_filter_node(node)

    return full_res_buffer
```

### Parameters and Typical Values
*   **Voxel Resolution:** $256 \times 256 \times 256$ [S1, unofficial].
*   **Cone Tracing Resolution:** Half screen resolution ($W/2 \times H/2$) [S1, unofficial].
*   **Extended Frustum Distance:** $D_{max} + \text{voxel size}$ [S1, unofficial].
*   **Normal Update Method:** `InterlockedMax` for dynamic, Average for static [S1, unofficial].

### Edge Cases and Pitfalls
*   **Lighting Artifacts:** Using `InterlockedMax()` for normals in small voxels causes incorrect light reflection compared to average normals [S1, unofficial].
*   **Upscaling Artifacts:** Half-resolution cone tracing creates visible artifacts at geometry edges when upscaled [S1, unofficial].
*   **Performance Drop:** Re-tracing all edge pixels at full resolution can significantly drop frame rate [S1, unofficial].
*   **Far Objects:** Limiting cone tracing distance may cause loss of sampling for far objects like the sky [S1, unofficial].

## Subtopics

### [Sparse Voxel Octree with Bricks](2354-sparse-voxel-octree-with-bricks.md)
Uses brick-based storage for better cache locality; choose when memory bandwidth is the bottleneck.

### [Naive Octree Node Representation](2445-naive-octree-node-representation.md)
Simple node structure without dynamic optimization; choose for static scenes or simpler implementations.

### [SVRaster](2464-svraster.md)
Rasterization-based voxelization; choose when GPU rasterization is preferred over ray-based voxelization.

## Operating characteristics
*   **Time Complexity:** Reduced per-frame cost for dynamic scenes. Static geometry is processed once. Dynamic updates are proportional to the number of dynamic objects and affected nodes [S1, unofficial].
*   **Memory Complexity:** Increased due to appending dynamic fragments and storing counters/flags [S1, unofficial].
*   **Measured Performance:** Faster than full re-voxelization. Half-resolution cone tracing with edge correction is faster than full-resolution cone tracing but may have quality trade-offs [S1, unofficial].
*   **Strengths:** Real-time performance for dynamic scenes; efficient use of GPU parallelism [S1, unofficial].
*   **Limitations:** Quality degradation from `InterlockedMax` normals and half-resolution upsampling; artifacts at geometry edges [S1, unofficial].
*   **When to Use:** Real-time applications with dynamic objects and indirect lighting requirements [S1, unofficial].
*   **When Not to Use:** Static scenes (use standard voxelization); high-quality offline rendering (use full-resolution cone tracing) [S1, unofficial].

## Background
The method was described by Simon Green in his blog post "Implementing Voxel Cone Tracing" in January 2013 [S1, unofficial]. It was developed as part of an experimental implementation of voxel cone tracing to achieve real-time indirect illumination [S1, unofficial]. The techniques were derived from practical experience optimizing the algorithm for dynamic scenes [S1, unofficial].

## Sources
[S1] Implementing Voxel Cone Tracing, http://simonstechblog.blogspot.com/2013/01/implementing-voxel-cone-tracing.html, fetched 2026-10-27 (unofficial)
