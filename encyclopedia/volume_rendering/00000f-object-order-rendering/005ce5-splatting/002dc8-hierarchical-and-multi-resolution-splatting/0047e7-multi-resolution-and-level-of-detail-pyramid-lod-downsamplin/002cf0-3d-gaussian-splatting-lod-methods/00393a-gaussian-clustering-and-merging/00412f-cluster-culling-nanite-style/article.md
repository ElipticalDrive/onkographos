# Cluster Culling (Nanite-style)

Also called: FClusterCullingCS, FClusterCullingResetCS

## Overview
Cluster Culling (Nanite-style) is a hierarchical frustum culling and Level of Detail (LOD) selection system for 3D Gaussian Splatting, designed to efficiently cull invisible or low-priority clusters from the rendering pipeline. It operates on a pre-computed hierarchy of Gaussian clusters, testing each cluster's bounding volume against the view frustum and calculating screen-space error to determine the appropriate LOD level. Unlike sibling methods that focus on the *merging* of Gaussians (e.g., [K-wise moment-matched merging](5440-k-wise-moment-matched-merging-with-l2-optimal-merging.md) or [Hierarchical Gaussian Merging](8358-hierarchical-gaussian-merging.md)), this method focuses on the *selection* and *culling* of those clusters for rendering, enabling performance scaling similar to Unreal Engine's Nanite system.

## How it works
The method consists of two primary compute shader stages: a reset stage to initialize buffers and a culling stage to determine visibility and LOD.

### Inputs and Outputs
**Inputs:**
- `ClusterBuffer`: A structured buffer containing `FGaussianGPUCluster` objects. Each cluster contains a bounding sphere (center and radius) and metadata indicating its LOD level and child cluster indices.
- `FrustumPlanes`: An array of 6 plane vectors ($\mathbf{n}, d$) defining the view frustum in world space.
- `LocalToWorld`: A $4 \times 4$ transformation matrix from local cluster space to world space.
- `WorldToClip`: A $4 \times 4$ projection matrix from world space to clip space.
- `CameraPosition`: The camera's position in world space.
- `ScreenHeight`: The height of the viewport in pixels.
- `ErrorThreshold`: The maximum allowable screen-space error (in pixels) for a cluster to be rendered at a lower LOD.
- `LODBias`: A bias added to the calculated screen-space error to favor lower LODs.
- `MaxDrawDistance`: The maximum distance from the camera at which clusters are considered for visibility.
- `LeafClusterCount`: The number of leaf clusters (raw Gaussians or base-level clusters) in the hierarchy.
- `ClusterCount`: The total number of clusters in the hierarchy.
- `UseLODRendering`: A flag indicating whether LOD rendering is enabled.
- `DebugForceLODLevel`: A debug parameter to force a specific LOD level (-1 for auto, 0 for leaf, 1+ for specific level).

**Outputs:**
- `VisibleClusterBuffer`: A buffer containing indices of visible clusters.
- `VisibleClusterCountBuffer`: A counter for the number of visible clusters.
- `IndirectDrawArgsBuffer`: Indirect draw arguments for the rendering engine.
- `ClusterVisibilityBitmap`: A bitmap indicating the visibility status of each cluster.
- `SelectedClusterBuffer`: A buffer of selected cluster indices for rendering.
- `LODClusterBuffer`: A buffer of clusters selected for specific LOD levels.
- `LODClusterCountBuffer`: A counter for clusters at each LOD level.
- `LODClusterSelectedBitmap`: A bitmap for LOD cluster selection.
- `LODSplatTotalBuffer`: Total number of splats (Gaussians) in visible LOD clusters.
- `LODSplatOutputCountBuffer`: Count of splats to be output.

### Data Structures
**FGaussianGPUCluster:**
- `Center`: `float3` (World space center of the bounding sphere).
- `Radius`: `float` (Radius of the bounding sphere in world space).
- `LODLevel`: `uint` (The LOD level of this cluster, 0 for leaf).
- `ChildIndices`: `uint[4]` (Indices of child clusters, if not a leaf).
- `SplatCount`: `uint` (Number of Gaussians in this cluster).

### Steps
1. **Reset Stage (`FClusterCullingResetCS`):**
   - Initialize `VisibleClusterCountBuffer` to 0.
   - Initialize `IndirectDrawArgsBuffer` with default values (e.g., 0 instances).
   - Clear `ClusterVisibilityBitmap` to 0.
   - Clear `SelectedClusterBuffer`, `LODClusterBuffer`, `LODClusterCountBuffer`, `LODClusterSelectedBitmap`, `LODSplatTotalBuffer`, and `LODSplatOutputCountBuffer`.

2. **Culling Stage (`FClusterCullingCS`):**
   - For each cluster `i` in `ClusterBuffer`:
     - If `DebugForceLODLevel` is not -1, set `LODLevel` to `DebugForceLODLevel`.
     - Calculate the cluster's world-space bounding sphere center $\mathbf{c}$ and radius $r$.
     - **Frustum Culling:**
       - Test the bounding sphere against each of the 6 frustum planes.
       - A sphere is inside the frustum if the distance from the sphere center to each plane is greater than or equal to the radius: $d_i = \mathbf{n}_i \cdot \mathbf{c} + d_i \ge r$ for all $i \in \{0, \dots, 5\}$.
       - If the sphere is outside any plane, mark the cluster as invisible and skip further processing.
     - **Distance Culling:**
       - Calculate the distance from the camera to the cluster center: $dist = \| \mathbf{c} - \mathbf{CameraPosition} \|$.
       - If $dist > MaxDrawDistance$, mark the cluster as invisible.
     - **LOD Selection:**
       - If the cluster is visible and `UseLODRendering` is enabled:
         - Project the cluster's bounding sphere to screen space to estimate its screen-space radius $r_{screen}$.
         - Calculate the screen-space error $E$ based on the cluster's size and distance. A common approximation is $E = \frac{r_{screen} \times ScaleFactor}{LODLevel + 1}$, where $ScaleFactor$ is a constant related to the Gaussian spread. [G] *Note: The exact formula for screen-space error is not provided in the sources; it typically involves the cluster's bounding sphere radius projected to screen space and compared against a threshold.*
         - If $E > ErrorThreshold + LODBias$, select a higher LOD level (coarser cluster) if available.
         - If no higher LOD is available or the error is within threshold, select the current cluster.
     - **Visibility Update:**
       - If the cluster is selected, append its index to `SelectedClusterBuffer`.
       - Update `ClusterVisibilityBitmap` to mark the cluster as visible.
       - If the cluster is a leaf, add its `SplatCount` to `LODSplatTotalBuffer`.
       - If the cluster is not a leaf, add its `SplatCount` to `LODSplatTotalBuffer` (assuming non-leaves are also rendered or their children are counted). [G] *Note: The sources do not specify if non-leaf clusters are rendered directly or if only leaves are rendered. Nanite-style systems typically render the selected LOD level, which may be a non-leaf cluster representing a group of Gaussians.*
   - Update `VisibleClusterCountBuffer` with the number of selected clusters.
   - Update `IndirectDrawArgsBuffer` with the total number of instances (clusters) and splats.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# FClusterCullingResetCS
def FClusterCullingResetCS(
    VisibleClusterCountBuffer,
    IndirectDrawArgsBuffer,
    ClusterVisibilityBitmap,
    SelectedClusterBuffer,
    LODClusterBuffer,
    LODClusterCountBuffer,
    LODClusterSelectedBitmap,
    LODSplatTotalBuffer,
    LODSplatOutputCountBuffer,
    ClusterVisibilityBitmapSize,
    LeafClusterCount,
    DebugForceLODLevel
):
    # Initialize counters
    VisibleClusterCountBuffer[0] = 0
    IndirectDrawArgsBuffer[0] = 0  # Draw count
    IndirectDrawArgsBuffer[1] = 0  # Instance count
    IndirectDrawArgsBuffer[2] = 0  # Start index
    IndirectDrawArgsBuffer[3] = 0  # Start offset

    # Clear bitmaps and buffers
    for i in range(ClusterVisibilityBitmapSize):
        ClusterVisibilityBitmap[i] = 0
    LODClusterSelectedBitmap[0] = 0
    LODSplatTotalBuffer[0] = 0
    LODSplatOutputCountBuffer[0] = 0

# FClusterCullingCS
def FClusterCullingCS(
    ClusterBuffer,
    VisibleClusterBuffer,
    VisibleClusterCountBuffer,
    IndirectDrawArgsBuffer,
    ClusterVisibilityBitmap,
    SelectedClusterBuffer,
    LODClusterBuffer,
    LODClusterCountBuffer,
    LODClusterSelectedBitmap,
    LODSplatTotalBuffer,
    LocalToWorld,
    WorldToClip,
    ClusterCount,
    LeafClusterCount,
    FrustumPlanes,
    CameraPosition,
    ScreenHeight,
    ErrorThreshold,
    LODBias,
    MaxDrawDistance,
    UseLODRendering,
    DebugForceLODLevel
):
    visible_count = 0
    total_splats = 0

    for i in range(ClusterCount):
        cluster = ClusterBuffer[i]

        # Debug force LOD
        if DebugForceLODLevel != -1:
            cluster.LODLevel = DebugForceLODLevel

        # Transform center to world space if not already
        # Note: ClusterBuffer is assumed to be in world space or transformed by LocalToWorld
        center = cluster.Center
        radius = cluster.Radius

        # Frustum Culling
        visible = True
        for plane in FrustumPlanes:
            # plane is (normal, distance)
            dist = dot(plane.normal, center) + plane.distance
            if dist < radius:
                visible = False
                break

        if not visible:
            continue

        # Distance Culling
        dist_to_camera = length(center - CameraPosition)
        if dist_to_camera > MaxDrawDistance:
            continue

        # LOD Selection
        lod_level = cluster.LODLevel
        if UseLODRendering and DebugForceLODLevel == -1:
            # Project radius to screen space
            # Approximate screen-space radius
            # Note: Exact projection formula not provided in sources
            # Assuming a simple perspective projection approximation
            clip_center = mul(WorldToClip, float4(center, 1.0))
            if clip_center.w > 0:
                ndc_radius = radius / clip_center.w * ScreenHeight / 2.0
                # Calculate error
                # Note: Exact error formula not provided in sources
                # Assuming error is proportional to screen-space radius and inversely proportional to LOD
                error = ndc_radius / (lod_level + 1)
                if error > ErrorThreshold + LODBias:
                    # Select higher LOD if available
                    # Note: Logic for selecting higher LOD cluster index not provided
                    # Assuming cluster.LODLevel is updated to a higher value
                    lod_level = lod_level + 1  # Placeholder for actual LOD selection logic

        # Mark as visible
        ClusterVisibilityBitmap[i] = 1
        SelectedClusterBuffer[visible_count] = i
        visible_count += 1

        # Count splats
        total_splats += cluster.SplatCount

    # Update output buffers
    VisibleClusterCountBuffer[0] = visible_count
    IndirectDrawArgsBuffer[1] = visible_count  # Instance count
    IndirectDrawArgsBuffer[0] = total_splats   # Draw count (if each cluster draws its splats)
    LODSplatTotalBuffer[0] = total_splats
```

### Parameters
- `FrustumPlanes`: Array of 6 plane vectors.
- `ErrorThreshold`: Threshold for screen-space error calculation. Typical value: Not given in the sources.
- `LODBias`: Bias applied to LOD selection. Typical value: Not given in the sources.
- `MaxDrawDistance`: Maximum distance for cluster visibility. Typical value: Not given in the sources.
- `DebugForceLODLevel`: Debug parameter to force a specific LOD level. Typical value: -1 (auto).
- `THREADGROUP_SIZE`: Compilation define set to 64 for `FClusterCullingCS`.

### Edge Cases and Pitfalls
- **Precision:** Frustum culling tests should use sufficient precision to avoid false negatives/positives for clusters on the frustum boundary.
- **Degenerate Input:** Clusters with zero radius or splat count should be handled gracefully (e.g., skipped or treated as invisible).
- **LOD Selection:** The exact formula for screen-space error and LOD selection is not provided in the sources. Implementers must derive this based on the specific Gaussian representation and desired visual quality.

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of clusters. Each cluster is processed once in the culling stage.
- **Memory Complexity:** $O(N)$ for storing cluster data and visibility buffers.
- **Performance:** Performance scales with the number of clusters and the complexity of the frustum culling test. The use of compute shaders allows for parallel processing of clusters.
- **Strengths:** Efficient culling of invisible clusters, adaptive LOD selection based on screen-space error, integration with hierarchical clustering structures.
- **Limitations:** Dependent on the quality of the pre-computed cluster hierarchy. LOD selection logic is not fully specified in the sources.
- **When to use:** When rendering large scenes with many Gaussian splats and requiring efficient frustum culling and LOD management.
- **When not to use:** For small scenes where the overhead of hierarchical culling is not justified.

## Background
Cluster Culling (Nanite-style) is inspired by Unreal Engine's Nanite virtualized geometry system, which uses hierarchical clustering and culling to render large amounts of geometry efficiently. In the context of 3D Gaussian Splatting, it adapts this concept to manage the rendering of Gaussian splats by clustering them and culling invisible or low-priority clusters. The method is implemented in the OpenSplat4D project [S1, unofficial], which provides a native 4D implementation of Gaussian splatting.

## Sources
[S1] YuanBaoSMadLab/OpenSplat4D, https://github.com/YuanBaoSMadLab/OpenSplat4D, fetched 2026-10-27 (unofficial)
