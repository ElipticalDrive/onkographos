# Octree Voxel Raycasting for Hover (Closest Active Voxel)

Also called: Octree Closest Hit Tracing

## Overview
This method performs raycasting against a Sparse Voxel Octree (SVO) to determine the closest intersection with an active voxel, enabling "hover" interactions in volume rendering. It differs from standard SVO ray tracing [2989-sparse-voxel-octree-ray-tracing.md] by specifically targeting the closest *active* leaf node (hit) while also tracking the closest *empty* space node for fallback or depth information. It serves as a specialized traversal for interaction rather than full volume integration.

## How it works
The method operates in a host-device loop. The host prepares ray data, dispatches a GPU kernel to traverse the octree, and downloads results. The GPU kernel performs the traversal, updating global state for the closest hit and empty space.

**Inputs:**
- `rays`: A buffer of `Ray` structs, size $W \times H$ (where $W, H$ are resolution dimensions).
- `octree`: The Sparse Voxel Octree structure, assumed to be accessible on the GPU.
- `resolution`: The width and height of the ray grid.

**Outputs:**
- `hits`: A buffer of `Hit` structs, size $W \times H$, containing traversal results.
- `host_hits`: A host-side `std::vector<Hit>` of size $W \times H$ for downloaded results.

**Data Structures:**
1.  **Ray**: Contains origin and direction vectors. Layout is not explicitly defined in sources, but standard ray representation is implied.
2.  **Hit**: Contains intersection data. Specific fields are not detailed in the provided sources, but `hitDistance` and `hitPosition`/`normal` are referenced in logic.
3.  **OctreeNode** (from [S2, unofficial]):
    - `isLeaf`: `bool`
    - `isActive`: `bool`
    - `position`: `glm::vec3` (Node center)
    - `size`: `glm::vec3` (Node dimensions)
    - `color`: `glm::vec3`
    - `children`: `OctreeNode*` array of size 8.

**Algorithm Steps:**

1.  **Initialization (Host):**
    - Create GPU buffers `rays` and `hits` of size `resolution * resolution` [S1, unofficial].
    - Initialize `closestHitDistance` to infinity, `closestEmptyDistance` to infinity, and `foundActiveVoxel` to false [S2, unofficial].
    - Reset `hoverInfo.hasHit` to false [S2, unofficial].

2.  **Ray Dispatch (Host):**
    - Populate the `rays` buffer with ray data for each pixel in the resolution grid [S1, unofficial].
    - Dispatch the GPU kernel `octree->trace_closest` with arguments: `rays->data()`, `hits->data()`, `resolution`, `resolution` [S1, unofficial].

3.  **GPU Traversal (Kernel):**
    - For each ray (thread):
        - Traverse the octree from the root.
        - If an active leaf node is intersected:
            - Compute `hitDistance` (distance from ray origin to intersection).
            - Compute `hitPosition` and `normal` [S2, unofficial].
            - If `hitDistance < closestHitDistance`:
                - Update `closestHitDistance = hitDistance`.
                - Update `closestHitPosition = hitPosition`.
                - Update `closestHitNormal = normal`.
                - Set `foundActiveVoxel = true`.
        - If an empty space node (non-active internal node or empty leaf) is intersected:
            - Compute `hitDistance` to the node boundary.
            - If `!foundActiveVoxel` AND `depth >= currentDepth` AND `hitDistance < closestEmptyDistance`:
                - Update `closestEmptyDistance = hitDistance`.
                - Update `hoverInfo` to point to this empty space node.
                - Set `hoverInfo.hasHit = true` [S2, unofficial].

4.  **Result Download (Host):**
    - Download `hits` buffer to `host_hits` using `hits->download(host_hits.data())` [S1, unofficial].

5.  **Post-Processing (Host):**
    - Check `foundActiveVoxel`. If true, the closest active voxel hit is recorded in `hits`. If false, `hoverInfo` may contain the closest empty space hit for fallback [S2, unofficial].

**Parameters:**
- `resolution`: Width and height of the ray grid. Typical values depend on display resolution (e.g., 1920x1080).
- `currentDepth`: The maximum depth of the octree traversal. Not given in the sources.

**Edge Cases:**
- **No Hit:** If no active voxel is intersected, `foundActiveVoxel` remains false. The method relies on `hoverInfo` for empty space feedback [S2, unofficial].
- **Precision:** Floating-point precision in distance comparisons may affect closest-hit determination. Not given in the sources.
- **Boundary Conditions:** Ray origins exactly on node boundaries may cause ambiguity. Not given in the sources.

## Operating characteristics
- **Time Complexity:** $O(N \cdot D)$, where $N$ is the number of rays ($W \times H$) and $D$ is the average depth of the octree traversal.
- **Memory Complexity:** $O(W \times H)$ for ray and hit buffers.
- **Strengths:** Efficient for interaction tasks requiring closest hit; leverages GPU parallelism.
- **Limitations:** Requires GPU memory for buffers; traversal depth depends on octree structure.
- **When to Use:** Hover interactions, picking, or closest-hit queries in SVO-based renderers.
- **When Not to Use:** Full volume rendering, ray marching with accumulation, or CPU-only environments.

## Background
This method is part of the SolidVoxelization project [S1, unofficial] and model_voxelizer [S2, unofficial], focusing on efficient voxel-based rendering and interaction. It builds on [Sparse Voxel Octree Ray Tracing](2989-sparse-voxel-octree-ray-tracing.md) by specializing the traversal for closest active voxel detection.

## Sources
[S1] Mike-Leo-Smith/SolidVoxelization, https://github.com/Mike-Leo-Smith/SolidVoxelization, fetched 2026-10-27 (unofficial)
[S2] GitHub - saurlax/model_voxelizer: A 3D model voxelization tool that ..., https://github.com/saurlax/model_voxelizer, fetched 2026-10-30 (unofficial)
