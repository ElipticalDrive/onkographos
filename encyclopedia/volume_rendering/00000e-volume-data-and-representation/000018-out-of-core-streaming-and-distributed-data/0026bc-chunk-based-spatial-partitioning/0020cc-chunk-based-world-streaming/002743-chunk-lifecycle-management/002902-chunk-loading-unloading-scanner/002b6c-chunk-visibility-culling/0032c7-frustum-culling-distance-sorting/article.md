# Frustum Culling & Distance Sorting

Also called: Chunk Visibility Culling with Distance Sorting

## Overview
This method filters visible chunks and sorts them by distance for rendering order [S1, unofficial]. It builds on [Chunk Visibility Culling](5031-chunk-visibility-culling.md) by adding a distance-based sort to the visible set, ensuring correct depth ordering for transparent or blended rendering passes [G]. It differs from [Octree Frustum Culling & LOD Selection](5771-octree-frustum-culling-lod-selection.md) by operating on flat chunk lists rather than hierarchical tree traversal, and from [Out-of-Core Frustum Clipping](6688-out-of-core-frustum-clipping.md) by not performing geometric clipping of chunk geometry, only visibility culling [G].

## How it works
**Inputs:** Camera view and projection matrices; list of active chunks; camera world position [G].
**Outputs:** Sorted list of visible chunks [S1, unofficial].

**Data Structures:**
*   `Chunk`: Contains world position fields `GetXWorldPos()` and `GetZWorldPos()` returning floating-point coordinates [S1, unofficial].
*   `Frustum`: Constructed from view and projection matrices. Provides method `IsChunkInFrustum(wx, wz)` which checks if a chunk's horizontal bounds intersect the frustum [S1, unofficial].
*   `m_LastCamChunkPos`: Tracks the camera's chunk position to avoid unnecessary sorting [S1, unofficial].

**Steps:**
1.  **Frustum Construction:** Create the view frustum from the current camera view and projection matrices [S1, unofficial].
2.  **Camera Chunk Tracking:** Compute the current camera chunk position [S1, unofficial].
3.  **Change Detection:** If the camera chunk position has changed since the last frame (compared against `m_LastCamChunkPos`), proceed to rebuild the ordered list [S1, unofficial].
4.  **Visibility Filtering:** Iterate through all chunks. For each chunk, check if it is ready (`IsMeshReady()`) and if it intersects the frustum using `frustum.IsChunkInFrustum(wx, wz)` [S1, unofficial]. Add visible chunks to a temporary list.
5.  **Distance Sorting:** Sort the visible chunks by distance from the camera. The sort uses a comparator that calculates squared distances in the XZ plane [S1, unofficial].

**Distance Calculation Formula:**
For a chunk $A$ with world position $(x_A, z_A)$ and camera position $(x_C, z_C)$:
$$ distSq_A = (x_A - x_C)^2 + (z_A - z_C)^2 $$
The comparator returns `true` if $distSq_A > distSq_B$, resulting in a descending sort order (farthest first) [S1, unofficial].

**Parameters:**
*   `m_LastCamChunkPos`: The previous camera chunk position [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **2D Distance:** The sorting uses only X and Z coordinates, ignoring Y. This assumes chunks are sorted on the horizontal plane [S1, unofficial].
*   **Precision:** Using squared distance avoids precision issues with square roots but may lead to ties for chunks at the same horizontal distance [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def UpdateChunkVisibility(view_matrix, proj_matrix, camera_world_pos, chunks, last_cam_chunk_pos, chunk_size=16):
    # Step 1: Create frustum from projection and view matrices
    frustum = ExtractFrustumPlanes(view_matrix, proj_matrix)

    # Step 2: Compute camera chunk position
    current_cam_chunk_pos = (int(camera_world_pos.x / chunk_size), int(camera_world_pos.z / chunk_size))

    # Step 3: If camera chunk changed, rebuild ordered list
    if current_cam_chunk_pos != last_cam_chunk_pos:
        visible_chunks = []

        # Step 4: Iterate chunks, check IsMeshReady() and frustum.IsChunkInFrustum(wx, wz)
        for chunk in chunks:
            if chunk.IsMeshReady():
                wx = chunk.GetXWorldPos()
                wz = chunk.GetZWorldPos()
                if frustum.IsChunkInFrustum(wx, wz):
                    visible_chunks.append(chunk)

        # Step 5: Sort chunks using std::sort with distance comparison
        visible_chunks.sort(key=lambda c: GetSquaredHorizontalDistance(c, camera_world_pos), reverse=True)

        last_cam_chunk_pos = current_cam_chunk_pos

    return visible_chunks, last_cam_chunk_pos

def ExtractFrustumPlanes(view_matrix, proj_matrix):
    # Standard operation: compute six clipping planes from combined view-projection matrix [G]
    return FrustumPlanes()

def GetSquaredHorizontalDistance(chunk, camera_pos):
    dx = chunk.GetXWorldPos() - camera_pos.x
    dz = chunk.GetZWorldPos() - camera_pos.z
    return dx * dx + dz * dz
```

## Subtopics

### Chunk Sorting by Distance
Sorting chunks based on their distance to the camera, likely for depth sorting or LOD selection, though the specific usage of the sorted list is not detailed in this snippet (only the sort logic is shown) [S1, unofficial]. It uses squared distance in the XZ plane, ignoring Y, which assumes chunks are sorted on the horizontal plane [S1, unofficial].

## Operating characteristics
**Time Complexity:** $O(N \log N)$ for sorting $N$ visible chunks [G]. The frustum check is $O(N)$ [G].
**Memory Complexity:** $O(N)$ to store the list of visible chunks [G].
**Strengths:** Simple to implement; ensures correct rendering order for transparent chunks [G].
**Limitations:** Ignores Y-axis distance, which may cause incorrect sorting for vertically stacked chunks; recalculates full sort on chunk change, which may be inefficient if chunk changes are frequent but minor [S1, unofficial].
**When to use:** When transparent chunk rendering requires strict back-to-front or front-to-back ordering and chunks are primarily distributed horizontally [G].
**When not to use:** When vertical sorting is critical or when chunk visibility changes frequently without camera movement [G].

## Background
It is used in real-time voxel rendering systems to manage chunk visibility and rendering order [S1, unofficial].

## Sources
[S1] adrienlabayle/VoxelEngine, https://github.com/adrienlabayle/VoxelEngine, fetched 2026-10-27 (unofficial)
