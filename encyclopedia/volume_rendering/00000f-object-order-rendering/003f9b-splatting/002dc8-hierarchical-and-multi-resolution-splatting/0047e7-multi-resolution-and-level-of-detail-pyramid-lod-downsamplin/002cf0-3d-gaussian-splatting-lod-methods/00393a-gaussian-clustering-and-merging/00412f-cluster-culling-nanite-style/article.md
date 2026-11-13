# Cluster Culling (Nanite-style)

Also called: FClusterCullingCS, FClusterCullingResetCS

## Overview
Cluster Culling (Nanite-style) is a hierarchical culling and Level-of-Detail (LOD) selection mechanism for 3D Gaussian Splatting that determines which clusters of Gaussians are visible and at what resolution they should be rendered [S1, unofficial]. It reduces rendering overhead in dense scenes by testing cluster bounding spheres against the view frustum, calculating screen-space error for LOD selection, and outputting a compact list of visible cluster indices with LOD flags [S1, unofficial]. It focuses on the selection and culling phase rather than the geometric merging algorithm itself, building on the hierarchical structure established by [Gaussian clustering and merging](8436-gaussian-clustering-and-merging.md) [G].

## How it works
The method operates in two distinct compute shader passes: a reset pass to initialize buffers and a culling pass to determine visibility and LOD [S1, unofficial].

### Inputs and Outputs
**Inputs:**
*   `ClusterBuffer`: A `StructuredBuffer<FGaussianGPUCluster>` containing cluster data [S1, unofficial].
*   `FrustumPlanes`: An array of 6 plane vectors defining the view frustum [S1, unofficial].
*   `LocalToWorld`: A $4 \times 4$ transformation matrix transforming cluster positions from local space to world space [S1, unofficial].
*   `WorldToClip`: A $4 \times 4$ transformation matrix transforming world positions to clip space [S1, unofficial].
*   `CameraPosition`: The camera's position in world space (`FVector3f`) [S1, unofficial].
*   `ScreenHeight`: The height of the viewport in pixels, used for screen-space error calculation [S1, unofficial].
*   `ErrorThreshold`: A float threshold for determining if a cluster's LOD is sufficient [S1, unofficial].
*   `LODBias`: A float bias applied to the LOD selection calculation [S1, unofficial].
*   `MaxDrawDistance`: A float maximum distance for cluster visibility [S1, unofficial].
*   `UseLODRendering`: A `uint32` flag indicating if LOD rendering is enabled [S1, unofficial].
*   `DebugForceLODLevel`: An `int32` debug parameter (-1 for auto, 0 for leaf, 1+ for specific level) [S1, unofficial].

**Outputs (UAVs):**
*   `VisibleClusterCountBuffer`: A `RWStructuredBuffer<uint>` storing the total count of visible clusters [S1, unofficial].
*   `IndirectDrawArgsBuffer`: A `RWStructuredBuffer<uint>` storing indirect draw arguments for the rendering engine [S1, unofficial].
*   `ClusterVisibilityBitmap`: A `RWStructuredBuffer<uint>` indicating the visibility status of all clusters [S1, unofficial].
*   `VisibleClusterBuffer`: A `RWStructuredBuffer<uint>` containing the indices of visible clusters [S1, unofficial].
*   `SelectedClusterBuffer`: A `RWStructuredBuffer<uint>` containing indices of clusters selected for rendering [S1, unofficial].
*   `LODClusterBuffer`: A `RWStructuredBuffer<uint>` containing indices of clusters selected for specific LOD levels [S1, unofficial].
*   `LODClusterCountBuffer`: A `RWStructuredBuffer<uint>` storing counts per LOD level [S1, unofficial].
*   `LODClusterSelectedBitmap`: A `RWStructuredBuffer<uint>` bitmap for LOD cluster selection [S1, unofficial].
*   `LODSplatTotalBuffer`: A `RWStructuredBuffer<uint>` storing the total number of splats in visible LOD clusters [S1, unofficial].
*   `LODSplatOutputCountBuffer`: A `RWStructuredBuffer<uint>` storing the output count of splats [S1, unofficial].

### Data Structures
The exact internal layout of `FGaussianGPUCluster` is Not given in the sources. It must contain at minimum a bounding sphere center and radius to support the frustum and distance culling steps described in the evidence [S1, unofficial].

### Steps
1.  **Reset Pass (FClusterCullingResetCS):** Initialize `VisibleClusterCountBuffer`, `IndirectDrawArgsBuffer`, `ClusterVisibilityBitmap`, `LODClusterCountBuffer`, `LODSplatTotalBuffer`, and `LODSplatOutputCountBuffer` to zero [S1, unofficial].
2.  **Culling Pass (FClusterCullingCS):** Process each cluster index $j$ from $0$ to `ClusterCount` - 1 [S1, unofficial].
3.  **Frustum Culling:** Test the cluster's bounding sphere against the 6 frustum planes in `FrustumPlanes` [S1, unofficial]. A sphere is inside the frustum if the distance from its center to each plane is greater than or equal to its radius for all planes [G].
4.  **Distance Culling:** Calculate the distance from `CameraPosition` to the cluster center [G]. If the distance exceeds `MaxDrawDistance`, cull the cluster [S1, unofficial].
5.  **LOD Selection:** Calculate screen-space error for LOD selection based on the projected bounding sphere size relative to `ErrorThreshold` [S1, unofficial]. Apply `LODBias` and `DebugForceLODLevel` overrides, then clamp to the valid LOD range [S1, unofficial].
6.  **Output Generation:** Write the cluster index to `VisibleClusterBuffer` and increment `VisibleClusterCountBuffer` atomically [S1, unofficial]. Set the corresponding bit in `ClusterVisibilityBitmap` to 1 [S1, unofficial]. Write the cluster index to `LODClusterBuffer` at an offset determined by `LODClusterCountBuffer[L]`, increment `LODClusterCountBuffer[L]` atomically, and set the corresponding bit in `LODClusterSelectedBitmap` [S1, unofficial]. Accumulate the cluster's splat count to `LODSplatTotalBuffer` and `LODSplatOutputCountBuffer` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
from typing import List, Tuple

# FClusterCullingResetCS
def FClusterCullingResetCS(
    visible_cluster_count_buffer: List[int],
    indirect_draw_args_buffer: List[int],
    cluster_visibility_bitmap: List[int],
    lod_cluster_count_buffer: List[int],
    lod_splat_total_buffer: List[int],
    lod_splat_output_count_buffer: List[int],
    cluster_visibility_bitmap_size: int,
    leaf_cluster_count: int,
    debug_force_lod_level: int
):
    visible_cluster_count_buffer[0] = 0
    indirect_draw_args_buffer[0] = 0
    lod_splat_total_buffer[0] = 0
    lod_splat_output_count_buffer[0] = 0
    for i in range(cluster_visibility_bitmap_size):
        cluster_visibility_bitmap[i] = 0
    for i in range(leaf_cluster_count):
        lod_cluster_count_buffer[i] = 0

# FClusterCullingCS
def FClusterCullingCS(
    cluster_buffer: List[dict],
    visible_cluster_buffer: List[int],
    visible_cluster_count_buffer: List[int],
    indirect_draw_args_buffer: List[int],
    cluster_visibility_bitmap: List[int],
    selected_cluster_buffer: List[int],
    lod_cluster_buffer: List[int],
    lod_cluster_count_buffer: List[int],
    lod_cluster_selected_bitmap: List[int],
    lod_splat_total_buffer: List[int],
    lod_splat_output_count_buffer: List[int],
    local_to_world: List[List[float]],
    world_to_clip: List[List[float]],
    cluster_count: int,
    leaf_cluster_count: int,
    frustum_planes: List[Tuple[float, float, float, float]],
    camera_position: Tuple[float, float, float],
    screen_height: float,
    error_threshold: float,
    lod_bias: float,
    max_draw_distance: float,
    use_lod_rendering: int,
    debug_force_lod_level: int
):
    for cluster_idx in range(cluster_count):
        cluster = cluster_buffer[cluster_idx]
        center = cluster['center']
        radius = cluster['radius']
        splat_count = cluster.get('splat_count', 1)

        # Transform center to world space
        center_world = [
            local_to_world[0][0]*center[0] + local_to_world[0][1]*center[1] + local_to_world[0][2]*center[2] + local_to_world[0][3],
            local_to_world[1][0]*center[0] + local_to_world[1][1]*center[1] + local_to_world[1][2]*center[2] + local_to_world[1][3],
            local_to_world[2][0]*center[0] + local_to_world[2][1]*center[1] + local_to_world[2][2]*center[2] + local_to_world[2][3]
        ]

        # Frustum Culling
        is_visible = True
        for plane in frustum_planes:
            dist = plane[0]*center_world[0] + plane[1]*center_world[1] + plane[2]*center_world[2] + plane[3]
            if dist < -radius:
                is_visible = False
                break
        if not is_visible:
            continue

        # Distance Culling
        dist_to_cam = math.sqrt((center_world[0]-camera_position[0])**2 + (center_world[1]-camera_position[1])**2 + (center_world[2]-camera_position[2])**2)
        if dist_to_cam > max_draw_distance:
            continue

        # LOD Selection
        lod_level = 0
        if use_lod_rendering:
            clip_pos = [
                world_to_clip[0][0]*center_world[0] + world_to_clip[0][1]*center_world[1] + world_to_clip[0][2]*center_world[2] + world_to_clip[0][3],
                world_to_clip[1][0]*center_world[0] + world_to_clip[1][1]*center_world[1] + world_to_clip[1][2]*center_world[2] + world_to_clip[1][3],
                world_to_clip[2][0]*center_world[0] + world_to_clip[2][1]*center_world[1] + world_to_clip[2][2]*center_world[2] + world_to_clip[2][3],
                world_to_clip[3][0]*center_world[0] + world_to_clip[3][1]*center_world[1] + world_to_clip[3][2]*center_world[2] + world_to_clip[3][3]
            ]
            if clip_pos[3] != 0:
                projected_radius = radius * (screen_height / 2.0) / abs(clip_pos[3])
                if projected_radius > 0:
                    lod_level = int(math.log2(projected_radius / error_threshold))
                else:
                    lod_level = 0
            else:
                lod_level = 0

            lod_level += int(lod_bias)
            if debug_force_lod_level != -1:
                lod_level = debug_force_lod_level
            if lod_level < 0:
                lod_level = 0
            if lod_level >= leaf_cluster_count:
                lod_level = leaf_cluster_count - 1

        # Output Visibility (simulated GPU atomic operations)
        count = visible_cluster_count_buffer[0]
        visible_cluster_count_buffer[0] = count + 1
        visible_cluster_buffer[count] = cluster_idx
        cluster_visibility_bitmap[cluster_idx] = 1

        # Output LOD Info
        lod_count = lod_cluster_count_buffer[lod_level]
        lod_cluster_count_buffer[lod_level] = lod_count + 1
        lod_cluster_buffer[lod_count] = cluster_idx
        lod_cluster_selected_bitmap[cluster_idx] = 1

        # Accumulate splat counts
        lod_splat_total_buffer[0] += splat_count
        lod_splat_output_count_buffer[0] += splat_count
```

### Parameters
*   `FrustumPlanes`: Array of 6 plane vectors defining the view frustum [S1, unofficial].
*   `ErrorThreshold`: Threshold for screen-space error calculation for LOD selection [S1, unofficial].
*   `LODBias`: Bias applied to LOD selection [S1, unofficial].
*   `MaxDrawDistance`: Maximum distance for cluster visibility [S1, unofficial].
*   `DebugForceLODLevel`: Debug parameter to force a specific LOD level (-1 = auto, 0 = leaf, 1+ = specific level) [S1, unofficial].
*   `THREADGROUP_SIZE`: Compilation define set to 64 for `FClusterCullingCS` [S1, unofficial].

### Edge Cases and Pitfalls
*   **Precision:** Floating-point precision in frustum plane tests can cause flickering at boundaries; a small epsilon should be added to distance comparisons [G].
*   **Degenerate Clusters:** Clusters with zero radius or opacity must be guarded against to avoid division by zero in LOD calculations [G].
*   **Atomic Contention:** High cluster counts can lead to contention on atomic operations for `VisibleClusterCountBuffer` and `LODClusterCountBuffer`; prefix sums or sorting may be required for optimal scaling [G].
*   **Frustum Plane Winding:** Frustum planes must be consistently oriented (normals pointing inward) for correct distance calculations [G].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of clusters, as each cluster is processed once in the culling pass [G].
*   **Memory Complexity:** $O(N)$ for storing visibility buffers and LOD counts [G].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales linearly with cluster count; performance is bounded by GPU memory bandwidth and atomic operation contention [G].
*   **Strengths:** Reduces rendering overhead by culling invisible clusters and selecting appropriate LODs; integrates directly with hierarchical clustering structures [S1, unofficial].
*   **Limitations:** Dependent on the quality of the hierarchical clustering; poor clustering leads to inefficient culling and suboptimal LOD transitions [G].
*   **When to Use:** Dense scenes with many Gaussians where frustum culling and LOD are necessary for performance [G].
*   **When Not to Use:** Small scenes with few Gaussians where the overhead of culling may not be justified [G].

## Background
Cluster Culling (Nanite-style) is inspired by Nanite's virtualized geometry system, which uses hierarchical clustering and culling to render massive amounts of geometry efficiently [G]. In the context of 3D Gaussian Splatting, it adapts these principles to manage the large number of Gaussian primitives [G]. The implementation details are derived from the OpenSplat4D project, which provides a native 4D implementation of Gaussian splatting with Niagara compute shaders [S1, unofficial].

## Sources
[S1] YuanBaoSMadLab/OpenSplat4D, https://github.com/YuanBaoSMadLab/OpenSplat4D, fetched 2026-10-27 (unofficial)
