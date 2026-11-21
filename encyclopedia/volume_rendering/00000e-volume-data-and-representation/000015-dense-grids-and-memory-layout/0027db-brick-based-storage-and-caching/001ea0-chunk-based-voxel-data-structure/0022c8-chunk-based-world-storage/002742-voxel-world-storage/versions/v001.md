# Voxel World Storage

Also called: VoxelWorld
## Overview
Voxel World Storage is a dense grid data structure that maintains block types for each chunk in world coordinates, enabling direct ray querying of the voxel space [S1, unofficial]. It solves the problem of selecting or interacting with specific voxels in a scene that lacks traditional mesh colliders by providing a coordinate-mapped lookup of block types [S1, unofficial]. It differs from its siblings [Chunk Loading and Unloading via Hashmap and Area of Interest](4766-chunk-loading-and-unloading-via-hashmap-and-area-of-interest.md) by focusing on the in-memory storage and coordinate translation for direct access rather than just the lifecycle management of chunks [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A ray defined by origin and direction, and a set of chunks containing block data.
- Output: A `VoxelRayHit` structure containing the solid voxel coordinates, its block type, the face normal and identifier of the crossing boundary, the empty voxel preceding the hit, and the distance to the hit.

**Data Structures:**
1.  **VoxelWorld**:
    -   Fields:
        -   `Min`: Vector3 defining the minimum world coordinates of the world limits box.
        -   `Max`: Vector3 defining the maximum world coordinates of the world limits box.
        -   `Chunks`: A collection of chunk objects, each containing a block type array.
    -   Layout: The `Min` and `Max` fields define the bounding box of all active chunks. Positions outside this box are treated as empty [S1, unofficial].

2.  **Chunk**:
    -   Fields:
        -   `BlockTypes`: A dense array of block types.
        -   Size: The sample scene uses 2 bytes per voxel [S1, unofficial].
        -   Total Memory: The sample scene uses 32 MiB for voxels, comprising 1,024 chunks [S1, unofficial].

**Algorithm (Voxel Ray Traversal):**
The traversal follows the Amanatides and Woo algorithm [S1, unofficial].

1.  **Initialization**:
    -   Calculate the voxel coordinates of the ray origin: $v_{start} = \lfloor origin \rfloor$.
    -   Determine the step direction for each axis: $step_x = \text{sign}(direction.x)$, etc.
    -   Calculate the t-maximum values ($t_{max}$) and t-delta values ($t_{delta}$) for crossing voxel boundaries in each axis.
    -   Initialize current voxel $v = v_{start}$.

2.  **Traversal Loop**:
    -   While the ray has not exceeded the maximum distance (default: 64 voxels) [S1, unofficial] and has not exceeded the world limits ($Min \le v \le Max$) [S1, unofficial]:
        -   Identify the axis with the smallest $t_{max}$.
        -   Update $t_{max}$ for that axis by adding $t_{delta}$.
        -   Advance the voxel coordinate $v$ by $step$ in that axis.
        -   Retrieve the block type at $v$ from the `VoxelWorld` using world-to-chunk coordinate translation.
        -   If the block type is empty, continue.
        -   If the block type is solid:
            -   Record the hit:
                -   `SolidVoxel`: $v$.
                -   `BlockType`: The retrieved type.
                -   `Normal`: The face normal corresponding to the axis crossed.
                -   `Face`: The face identifier.
                -   `EmptyVoxel`: The previous voxel coordinate.
                -   `Distance`: The current traversal distance.
            -   Return `VoxelRayHit`.

3.  **Edge Cases**:
    -   **World Limits**: The ray stops if it exits the box defined by `Min` and `Max` [S1, unofficial].
    -   **Max Distance**: The ray stops if the distance exceeds the default 64 voxels [S1, unofficial].
    -   **Edges/Corners**: At an edge or corner, the ray crosses X first, then Y, then Z [S1, unofficial]. This ordering determines the face normal when multiple boundaries are crossed simultaneously.
    -   **Outside World**: A position outside all chunks is treated as empty [S1, unofficial].

**Pitfalls:**
-   Changing a chunk's block type array after `AddChunk` is permitted to allow in-place updates [S1, unofficial].
-   Precision issues with floating-point ray origins should be handled, though specific mitigation strategies are not detailed beyond noting that previous precision notes were "too strong" [S1, unofficial].

## Subtopics

### [libvxl Map Storage and Access](4145-libvxl-map-storage-and-access.md)
A sparse voxel map storage system for 'Ace of Spades' maps using a compressed internal format. It supports floating block detection and differs by maintaining an in-memory chunked structure for direct access rather than just serialization.

## Operating characteristics
-   **Time Complexity**: Ray traversal is $O(d)$ where $d$ is the distance to the hit or max distance, as it visits each voxel along the ray path [S1, unofficial].
-   **Memory Complexity**: $O(N)$ where $N$ is the total number of voxels. The sample scene uses 32 MiB for 1,024 chunks with 2 bytes per voxel [S1, unofficial].
-   **Strengths**: Direct access to block types without mesh colliders; efficient for ray-based selection [S1, unofficial].
-   **Limitations**: Dense storage requires significant memory for large worlds; traversal depth is limited by max distance [S1, unofficial].

## Background
Voxel World Storage was introduced in the Elysian-Outpost project to enable voxel ray querying for selecting voxels at the screen center, replacing the legacy Physics.Raycast on mesh colliders [S1, unofficial]. It was part of the M1 milestone port, adding features like the voxel ray and selection box pipeline [S1, unofficial].

## Sources
[S1] feat(#40): Find the voxel at the screen center with a voxel ray, https://github.com/nicolas-goyon/Elysian-Outpost/pull/113, fetched 2026-10-27 (unofficial)
