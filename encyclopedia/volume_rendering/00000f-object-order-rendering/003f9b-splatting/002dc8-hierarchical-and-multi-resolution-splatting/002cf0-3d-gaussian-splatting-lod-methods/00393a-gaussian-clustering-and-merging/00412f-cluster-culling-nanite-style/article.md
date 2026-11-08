# Cluster Culling (Nanite-style)

Also called: FClusterCullingCS, FClusterCullingResetCS

## Overview
Cluster Culling is a hierarchical frustum culling and Level-of-Detail (LOD) selection mechanism for 3D Gaussian Splatting, designed to emulate the spatial clustering efficiency of Nanite [S1, unofficial]. It operates on pre-computed clusters of Gaussians, testing their bounding volumes against the view frustum and selecting the appropriate LOD level based on screen-space error, thereby reducing the number of draw calls and splats processed per frame [S1, unofficial]. It differs from sibling methods by relying on a fixed, pre-built hierarchical cluster structure and GPU-driven indirect draw argument generation rather than dynamic merging or runtime LOD generation [S1, unofficial].

## How it works
The method consists of two primary compute shader passes: a reset pass to initialize buffers and a culling pass to determine visibility and LOD.

### Inputs and Outputs
**Inputs:**
*   **Cluster Data:** A structured buffer containing cluster metadata. Each cluster typically includes a bounding sphere (center and radius) and references to its child splats or sub-clusters [S1, unofficial].
*   **Camera State:** View-to-Clip matrix ($W_{clip}$), Local-to-World matrix ($W_{local}$), camera position, screen height, and frustum planes [S1, unofficial].
*   **Parameters:** Frustum planes (6 vectors), error threshold, LOD bias, max draw distance, and debug flags [S1, unofficial].

**Outputs:**
*   **Visible Clusters:** A list of cluster indices that pass culling, along with their selected LOD level [S1, unofficial].
*   **Indirect Draw Arguments:** Buffers containing the count and instance offsets for the rendering engine [S1, unofficial].
*   **Visibility Bitmaps:** Bitmasks indicating which clusters and LOD levels are active [S1, unofficial].

### Data Structures
The implementation relies on specific buffer layouts defined in the source [S1, unofficial].

**FGaussianGPUCluster** (implied from context):
*   `Center`: `float3` (Cluster center position)
*   `Radius`: `float` (Bounding sphere radius)
*   `LODLevels`: Array or structure defining child pointers or splat counts per LOD level
*   `Parent/Child Indices`: Indices for hierarchical traversal

**FClusterCullingResetCS Parameters:**
*   `VisibleClusterCountBuffer`: `RWStructuredBuffer<uint>`
*   `IndirectDrawArgsBuffer`: `RWStructuredBuffer<uint>`
*   `ClusterVisibilityBitmap`: `RWStructuredBuffer<uint>`
*   `SelectedClusterBuffer`: `RWStructuredBuffer<uint>`
*   `LODClusterBuffer`: `RWStructuredBuffer<uint>`
*   `LODClusterCountBuffer`: `RWStructuredBuffer<uint>`
*   `LODClusterSelectedBitmap`: `RWStructuredBuffer<uint>`
*   `LODSplatTotalBuffer`: `RWStructuredBuffer<uint>`
*   `LODSplatOutputCountBuffer`: `RWStructuredBuffer<uint>`
*   `ClusterVisibilityBitmapSize`: `uint32`
*   `LeafClusterCount`: `uint32`
*   `DebugForceLODLevel`: `int32`

**FClusterCullingCS Parameters:**
*   `ClusterBuffer`: `SRV StructuredBuffer<FGaussianGPUCluster>`
*   `VisibleClusterBuffer`: `UAV RWStructuredBuffer<uint>`
*   `VisibleClusterCountBuffer`: `UAV RWStructuredBuffer<uint>`
*   `IndirectDrawArgsBuffer`: `UAV RWStructuredBuffer<uint>`
*   `ClusterVisibilityBitmap`: `UAV RWStructuredBuffer<uint>`
*   `SelectedClusterBuffer`: `UAV RWStructuredBuffer<uint>`
*   `LODClusterBuffer`: `UAV RWStructuredBuffer<uint>`
*   `LODClusterCountBuffer`: `UAV RWStructuredBuffer<uint>`
*   `LODClusterSelectedBitmap`: `UAV RWStructuredBuffer<uint>`
*   `LODSplatTotalBuffer`: `UAV RWStructuredBuffer<uint>`
*   `LODSplatOutputCountBuffer`: `UAV RWStructuredBuffer<uint>`
*   `LocalToWorld`: `float4x4`
*   `WorldToClip`: `float4x4`
*   `ClusterCount`: `uint32`
*   `LeafClusterCount`: `uint32`
*   `FrustumPlanes`: `float4[6]` (Plane normals and distances)
*   `CameraPosition`: `float3`
*   `ScreenHeight`: `float`
*   `ErrorThreshold`: `float`
*   `LODBias`: `float`
*   `MaxDrawDistance`: `float`
*   `UseLODRendering`: `uint32`
*   `DebugForceLODLevel`: `int32`

### Steps

1.  **Reset Buffers (FClusterCullingResetCS):**
    *   Initialize `VisibleClusterCountBuffer` to 0.
    *   Initialize `IndirectDrawArgsBuffer` to 0.
    *   Clear `ClusterVisibilityBitmap`, `SelectedClusterBuffer`, `LODClusterBuffer`, `LODClusterCountBuffer`, `LODClusterSelectedBitmap`, `LODSplatTotalBuffer`, and `LODSplatOutputCountBuffer` to 0 [S1, unofficial].

2.  **Culling and LOD Selection (FClusterCullingCS):**
    *   Each thread processes one cluster.
    *   **Frustum Culling:** Test the cluster's bounding sphere against the 6 frustum planes. If the sphere is entirely outside any plane, the cluster is culled [S1, unofficial].
    *   **Distance Culling:** Calculate the distance from the camera position to the cluster center. If this distance exceeds `MaxDrawDistance`, the cluster is culled [S1, unofficial].
    *   **LOD Selection:** If the cluster is visible, determine the appropriate LOD level.
        *   Calculate screen-space error or projected size.
        *   Apply `LODBias` to the error calculation.
        *   Compare against `ErrorThreshold`.
        *   If `DebugForceLODLevel` is not -1, force the specified LOD level [S1, unofficial].
    *   **Output:**
        *   Write the cluster index to `VisibleClusterBuffer`.
        *   Atomically increment `VisibleClusterCountBuffer`.
        *   Write the selected LOD level to `LODClusterBuffer`.
        *   Update `LODClusterCountBuffer` and `LODSplatOutputCountBuffer` based on the number of splats in the selected LOD [S1, unofficial].
        *   Set bits in `ClusterVisibilityBitmap` and `LODClusterSelectedBitmap` [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# FClusterCullingResetCS
kernel FClusterCullingResetCS(
    RWStructuredBuffer<uint> VisibleClusterCountBuffer,
    RWStructuredBuffer<uint> IndirectDrawArgsBuffer,
    RWStructuredBuffer<uint> ClusterVisibilityBitmap,
    RWStructuredBuffer<uint> SelectedClusterBuffer,
    RWStructuredBuffer<uint> LODClusterBuffer,
    RWStructuredBuffer<uint> LODClusterCountBuffer,
    RWStructuredBuffer<uint> LODClusterSelectedBitmap,
    RWStructuredBuffer<uint> LODSplatTotalBuffer,
    RWStructuredBuffer<uint> LODSplatOutputCountBuffer,
    uint ClusterVisibilityBitmapSize,
    uint LeafClusterCount,
    int DebugForceLODLevel
) {
    uint idx = DispatchThreadID.x;
    
    // Reset counters
    VisibleClusterCountBuffer[0] = 0;
    IndirectDrawArgsBuffer[0] = 0;
    IndirectDrawArgsBuffer[1] = 0;
    IndirectDrawArgsBuffer[2] = 0;
    IndirectDrawArgsBuffer[3] = 0;
    
    // Clear bitmaps and buffers
    for (uint i = 0; i < ClusterVisibilityBitmapSize; i++) {
        ClusterVisibilityBitmap[i] = 0;
        LODClusterSelectedBitmap[i] = 0;
    }
    
    for (uint i = 0; i < LeafClusterCount; i++) {
        SelectedClusterBuffer[i] = 0;
        LODClusterBuffer[i] = 0;
        LODClusterCountBuffer[i] = 0;
        LODSplatTotalBuffer[i] = 0;
        LODSplatOutputCountBuffer[i] = 0;
    }
}

# FClusterCullingCS
kernel FClusterCullingCS(
    StructuredBuffer<FGaussianGPUCluster> ClusterBuffer,
    RWStructuredBuffer<uint> VisibleClusterBuffer,
    RWStructuredBuffer<uint> VisibleClusterCountBuffer,
    RWStructuredBuffer<uint> IndirectDrawArgsBuffer,
    RWStructuredBuffer<uint> ClusterVisibilityBitmap,
    RWStructuredBuffer<uint> SelectedClusterBuffer,
    RWStructuredBuffer<uint> LODClusterBuffer,
    RWStructuredBuffer<uint> LODClusterCountBuffer,
    RWStructuredBuffer<uint> LODClusterSelectedBitmap,
    RWStructuredBuffer<uint> LODSplatTotalBuffer,
    RWStructuredBuffer<uint> LODSplatOutputCountBuffer,
    float4x4 LocalToWorld,
    float4x4 WorldToClip,
    uint ClusterCount,
    uint LeafClusterCount,
    float4 FrustumPlanes[6],
    float3 CameraPosition,
    float ScreenHeight,
    float ErrorThreshold,
    float LODBias,
    float MaxDrawDistance,
    uint UseLODRendering,
    int DebugForceLODLevel
) {
    uint cluster_idx = DispatchThreadID.x;
    
    if (cluster_idx >= ClusterCount) return;
    
    FGaussianGPUCluster cluster = ClusterBuffer[cluster_idx];
    
    // Transform cluster center to world space
    float3 center_world = mul(LocalToWorld, float4(cluster.Center, 1.0)).xyz;
    
    // Distance culling
    float dist = distance(center_world, CameraPosition);
    if (dist > MaxDrawDistance) return;
    
    // Frustum culling
    bool inside = true;
    for (int i = 0; i < 6; i++) {
        // Plane equation: dot(normal, point) + distance <= 0 for inside
        // Assuming FrustumPlanes[i] = (normal, distance)
        float dist_to_plane = dot(FrustumPlanes[i].xyz, center_world) + FrustumPlanes[i].w;
        if (dist_to_plane > cluster.Radius) {
            inside = false;
            break;
        }
    }
    
    if (!inside) return;
    
    // LOD Selection
    int selected_lod = 0;
    if (DebugForceLODLevel != -1) {
        selected_lod = DebugForceLODLevel;
    } else if (UseLODRendering) {
        // Calculate screen-space error or projected size
        // Project center to clip space
        float4 clip_pos = mul(WorldToClip, float4(center_world, 1.0));
        float ndc_z = clip_pos.z / clip_pos.w;
        
        // Approximate screen space radius
        // Note: Exact formula depends on projection matrix specifics
        float screen_radius = cluster.Radius * ScreenHeight / max(ndc_z, 0.001);
        
        // Determine LOD based on error threshold and bias
        // Simplified logic: select lowest LOD where error < threshold
        for (int lod = 0; lod < cluster.MaxLODLevel; lod++) {
            float lod_error = cluster.LODErrors[lod] * (1.0 + LODBias);
            if (lod_error < ErrorThreshold) {
                selected_lod = lod;
                break;
            }
        }
    } else {
        selected_lod = 0; // Default to leaf or base LOD
    }
    
    // Output visible cluster
    uint count = InterlockedAdd(VisibleClusterCountBuffer[0], 1);
    VisibleClusterBuffer[count] = cluster_idx;
    
    // Output LOD info
    LODClusterBuffer[count] = selected_lod;
    
    // Update LOD-specific counters
    // Note: Implementation details for LODSplatTotalBuffer and LODSplatOutputCountBuffer
    // depend on the specific cluster structure. Assuming cluster stores splat counts per LOD.
    uint splat_count = cluster.SplatCounts[selected_lod];
    LODSplatTotalBuffer[cluster_idx] = splat_count;
    LODSplatOutputCountBuffer[cluster_idx] = splat_count;
    
    // Update bitmaps
    ClusterVisibilityBitmap[cluster_idx / 32] |= (1 << (cluster_idx % 32));
    LODClusterSelectedBitmap[cluster_idx / 32] |= (1 << (cluster_idx % 32));
    
    // Update Indirect Draw Args
    // Assuming IndirectDrawArgsBuffer is structured as [Count, StartIndex, ...]
    // This is a simplified update; actual implementation may vary
    InterlockedAdd(IndirectDrawArgsBuffer[0], splat_count);
}
```

### Parameters
*   **FrustumPlanes:** Array of 6 plane vectors defining the view frustum [S1, unofficial].
*   **ErrorThreshold:** Threshold for screen-space error calculation for LOD selection [S1, unofficial].
*   **LODBias:** Bias applied to LOD selection [S1, unofficial].
*   **MaxDrawDistance:** Maximum distance for cluster visibility [S1, unofficial].
*   **DebugForceLODLevel:** Debug parameter to force a specific LOD level (-1 = auto, 0 = leaf, 1+ = specific level) [S1, unofficial].
*   **THREADGROUP_SIZE:** Compilation define set to 64 for FClusterCullingCS [S1, unofficial].

### Edge Cases and Pitfalls
*   **Precision:** Frustum culling tests should use robust plane-sphere intersection tests to avoid artifacts near the frustum boundaries [S1, unofficial].
*   **Degenerate Input:** Clusters with zero radius or invalid bounding spheres should be handled gracefully, potentially by culling them or treating them as points [S1, unofficial].
*   **LOD Bias:** Incorrect LOD bias can lead to excessive detail (performance hit) or excessive simplification (visual artifacts) [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(C)$, where $C$ is the number of clusters. Each cluster is processed in constant time by a single thread [S1, unofficial].
*   **Memory Complexity:** $O(C)$ for cluster data and output buffers [S1, unofficial].
*   **Performance:** Highly efficient on GPU due to parallel processing of clusters. The use of indirect draw calls minimizes CPU-GPU synchronization overhead [S1, unofficial].
*   **Strengths:** Reduces draw calls significantly, especially for large scenes with many Gaussians. Hierarchical culling allows for efficient LOD selection [S1, unofficial].
*   **Limitations:** Requires pre-computed cluster hierarchy. LOD selection is based on screen-space error, which may not always be optimal for all visual qualities [S1, unofficial].

## Background
Cluster Culling (Nanite-style) is an adaptation of Nanite's virtualized geometry system to 3D Gaussian Splatting. It was implemented in the OpenSplat4D project, which provides a native 4D implementation of Gaussian Splatting [S1, unofficial]. The method leverages the hierarchical clustering and culling techniques to manage the large number of Gaussians efficiently [S1, unofficial].

## Sources
[S1] YuanBaoSMadLab/OpenSplat4D, https://github.com/YuanBaoSMadLab/OpenSplat4D, fetched 2026-10-27 (unofficial)
