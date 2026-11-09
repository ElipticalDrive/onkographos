# Dynamic Update Optimization

Also called: Dynamic Voxel Cone Tracing Optimization, Dynamic Scene Voxelization Speedup

## Overview
Dynamic Update Optimization is a set of techniques for accelerating indirect illumination calculations in dynamic scenes using Sparse Voxel Octrees (SVO). It solves the performance bottleneck of re-voxelizing and re-filtering the entire scene every frame by separating static and dynamic geometry, optimizing the voxelization process, and reducing the resolution of expensive cone tracing passes. It sets itself apart from static SVO methods by handling time-varying data without rebuilding the entire hierarchy, and from naive dynamic updates by using specific hardware primitives and resolution-adaptive shading.

## How it works
The method consists of four primary optimizations applied during the indirect illumination pipeline.

### Inputs and Outputs
*   **Inputs:** Scene geometry (static and dynamic), camera parameters (position, view/projection matrices, field of view), maximum cone tracing distance.
*   **Outputs:** Indirect lighting contribution per pixel (color).

### Data Structures
The SVO node structure is modified to support dynamic updates:
*   **Static/Dynamic Flag:** A bit flag in the octree node identifies whether the node contains static or dynamic data [S1, unofficial].
*   **Voxel Normal Counter:** A 1-byte counter is stored alongside the normal vector in the voxel data structure to facilitate atomic averaging for static geometry [S1, unofficial].
*   **Dynamic Buffer:** Dynamic voxel fragments are appended to the end of the octree node buffer, allowing for easy clearing without restructuring the static tree [S1, unofficial].
*   **Neighbor Offsets:** Static octree nodes maintain neighbor offsets that point to dynamic nodes, which must be reset when dynamic data changes [S1, unofficial].

### Steps

1.  **Scene Initialization (Static Pre-computation)**
    *   Perform the standard 5-step voxelization and filtering pipeline for all static geometry once at startup [S1, unofficial].
    *   Store static voxels in the octree. Mark them with the static bit flag [S1, unofficial].

2.  **Per-Frame Dynamic Update**
    *   Identify voxels affected by dynamic objects [S1, unofficial].
    *   Voxelize only the dynamic geometry for the current frame [S1, unofficial].
    *   Append dynamic voxel fragments to the end of the octree node buffer [S1, unofficial].
    *   Reset static octree node neighbor offsets pointing to dynamic nodes [S1, unofficial].
    *   Identify static nodes affected by dynamic voxels from the previous and current frames. This is done by dispatching threads for all nodes and queuing affected node indices into a buffer [S1, unofficial].
    *   Re-filter only the identified affected static nodes [S1, unofficial].

3.  **Optimized Dynamic Voxelization**
    *   Use `InterlockedMax()` to compute diffuse color and normal for dynamic voxels. This is faster than atomic average operations [S1, unofficial].
    *   **Constraint:** Do not use `InterlockedMax()` for normals in dynamic geometry if voxel volume is small (e.g., $256 \times 256 \times 256$), as it causes lighting artifacts (incorrect reflection direction) compared to average normals [S1, unofficial].
    *   **Hybrid Approach:** Use `InterlockedMax()` for normals in dynamic geometry only if quality degradation is acceptable; otherwise, keep static geometry computing average normals. The 1-byte counter in the voxel structure supports the average calculation for static data [S1, unofficial].

4.  **View-Frustum Culling for Direct Lighting Injection**
    *   Calculate an extended view frustum from the current camera to cull light injection [S1, unofficial].
    *   The extended frustum is created by moving the camera backward and increasing the far plane using the same field of view [S1, unofficial].
    *   The extended distance is calculated as:
        $$ D_{ext} = D_{max\_cone} + D_{voxel\_filter} $$
        Where $D_{max\_cone}$ is the maximum cone tracing distance and $D_{voxel\_filter}$ is the filtered voxel size [S1, unofficial].
    *   Only voxels within this extended frustum are filtered with direct lighting [S1, unofficial].

5.  **Half-Resolution Cone Tracing with Edge Repair**
    *   Perform the cone tracing pass at half screen resolution ($W/2 \times H/2$) [S1, unofficial].
    *   Upscale the result to full resolution using bilinear filtering [S1, unofficial].
    *   **Artifact Handling:** Bilinear upscaling causes visible artifacts at geometry edges [S1, unofficial].
    *   **Edge Detection:** Use an edge detection filter on the depth buffer to identify pixels at geometry boundaries [S1, unofficial].
    *   **Re-computation:** Perform cone tracing at full resolution only for the identified edge pixels [S1, unofficial].
    *   **Alternative (if performance is critical):** Blur the edge pixels by averaging with neighbors instead of re-computing cone tracing. This is faster but yields lower quality [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dynamic_update_optimization(scene, camera, frame):
    # Step 1: Initialization (One-time)
    if frame == 0:
        voxelize_static_geometry(scene.static_objects)
        filter_static_octree()
        mark_nodes_as_static()
    
    # Step 2: Per-Frame Dynamic Handling
    dynamic_voxels = voxelize_dynamic_geometry(scene.dynamic_objects)
    append_to_dynamic_buffer(dynamic_voxels)
    reset_static_neighbor_offsets_to_dynamic()
    
    # Identify affected static nodes
    affected_static_nodes = find_affected_static_nodes(
        previous_dynamic_voxels=scene.last_frame_dynamic_voxels,
        current_dynamic_voxels=dynamic_voxels
    )
    re_filter_nodes(affected_static_nodes)
    
    # Step 3: Optimized Voxelization Logic (Inside Voxelization Kernel)
    def update_dynamic_voxel(normal, color):
        # Use InterlockedMax for speed
        atomic_interlocked_max(normal_buffer, normal)
        atomic_interlocked_max(color_buffer, color)
        # Note: Static voxels use atomic_average with 1-byte counter
        pass

    # Step 4: Frustum Culling for Direct Lighting
    max_cone_dist = get_max_cone_tracing_distance()
    voxel_filter_size = get_filtered_voxel_size()
    extended_far_plane = camera.far_plane + max_cone_dist + voxel_filter_size
    
    # Move camera backward to create extended frustum
    extended_camera = camera.copy()
    extended_camera.position -= camera.forward * (max_cone_dist + voxel_filter_size)
    extended_camera.far_plane = extended_far_plane
    
    cull_voxels_for_direct_lighting(extended_camera)
    
    # Step 5: Cone Tracing with Resolution Adaptation
    # Pass 1: Half Resolution
    half_res_color = cone_trace(camera, resolution=(screen_w/2, screen_h/2))
    
    # Edge Detection
    depth_buffer = get_depth_buffer()
    edge_mask = detect_edges(depth_buffer)
    
    # Pass 2: Full Resolution for Edges
    full_res_color = cone_trace(camera, resolution=(screen_w, screen_h))
    
    # Composite
    final_color = lerp(half_res_color, full_res_color, edge_mask)
    
    # Alternative: Blur instead of re-trace
    # final_color = blur(half_res_color, edge_mask)
    
    return final_color
```

### Parameters
*   **Voxelization Resolution:** Not given in the sources.
*   **Cone Tracing Distance:** Limited in the demo; exact value not given [S1, unofficial].
*   **Upscaling Method:** Bilinear filtering [S1, unofficial].
*   **Edge Detection Filter:** Not specified [S1, unofficial].

### Edge Cases and Pitfalls
*   **Small Voxel Volume:** Using `InterlockedMax()` for normals in small voxels (e.g., $256^3$) causes visible lighting artifacts (incorrect reflections) [S1, unofficial].
*   **Far Objects:** Limiting cone tracing distance may lose the ability to sample far objects like the sky [S1, unofficial].
*   **Performance Drop:** Re-computing cone tracing at full resolution for edge pixels causes a significant frame rate drop [S1, unofficial].

## Subtopics

### [Priority Queue Based Field Update](7430-priority-queue-based-field-update.md)
Ensures coarse-to-fine update ordering for interactive sculpting, preventing visual artifacts.

### [Sparse Voxel Octree Memory Management and Texture Upload](11147-sparse-voxel-octree-memory-management-and-texture-upload.md)
Manages flat integer array layout and incremental 256x256 slice texture uploads for dynamic data.

## Operating characteristics
*   **Time Complexity:** Reduced per-frame cost by only processing dynamic geometry and affected static nodes. Cone tracing is reduced by half-resolution processing, though edge repair adds overhead [S1, unofficial].
*   **Memory Complexity:** SVO structure requires extra space for dynamic buffer and 1-byte normal counter [S1, unofficial].
*   **Performance:** Faster than full re-voxelization. Edge repair at full resolution is costly; blurring is faster but lower quality [S1, unofficial].
*   **Strengths:** Handles dynamic scenes efficiently; reduces GPU load for indirect lighting.
*   **Limitations:** Artifacts from `InterlockedMax()` on normals; artifacts from half-res upscaling; potential loss of far-field lighting [S1, unofficial].

## Background
This method was described by Simon O'Raifeartaigh in a blog post detailing the implementation of Voxel Cone Tracing. It addresses the performance challenges of real-time indirect illumination in dynamic scenes.

## Sources
[S1] Implementing Voxel Cone Tracing, http://simonstechblog.blogspot.com/2013/01/implementing-voxel-cone-tracing.html, fetched 2026-10-27 (unofficial)
