# Voxel Block Interaction (Raycast Hit Detection)
Also called: FindSolidBlock
## Overview
This method identifies the specific solid voxel block intersected by a player's interaction raycast by evaluating the hit location's integer grid coordinates and its six immediate neighbors. It solves the problem of discrete block selection in a chunked voxel world where a continuous raycast hit point may lie on the boundary between blocks or within an empty space adjacent to a solid one. It differs from its parent, [Voxel Raycasting for Block Interaction](3780-voxel-raycasting-for-block-interaction.md), which steps along the ray to find the first solid voxel, by instead performing a localized search around a known hit location to determine the precise block for breaking or placing.
## How it works
**Inputs:**
- `HitLocation`: The world-space 3D vector where the interaction ray intersects the voxel geometry.
- `S`: The side length of a single voxel block (BlockSize).
- `ChunkSizeZ`: The vertical dimension of a chunk in blocks.
- `ChunkCache`: A data structure mapping chunk indices to block data arrays.
- `GlobalToLocal`: A helper function converting world block coordinates to chunk index and local block coordinates.

**Outputs:**
- `OutChunkIndex`: The 2D integer index of the chunk containing the target block.
- `OutLocalBlock`: The 3D integer coordinates of the block within that chunk.
- `bFound`: A boolean indicating whether a solid block was successfully identified.

**Data Structures:**
- `FIntPoint`: A 2D integer vector representing the chunk index $(CI_x, CI_y)$.
- `FIntVector`: A 3D integer vector representing local block coordinates $(LB_x, LB_y, LB_z)$.
- `ChunkCache`: A map or array where keys are `FIntPoint` and values are pointers to block data arrays. A `null` value indicates an unloaded or empty chunk.

**Parameters:**
- `S`: Block size. Typical value depends on the world scale, accessed via `AWorldMeshActor::BlockSize` [S1, unofficial].

**Algorithm Steps:**
1.  **Calculate Center Grid Coordinates:** Determine the integer grid coordinates of the hit location by dividing by the block size and flooring:
    $$
    \begin{aligned}
    CenterX &= \lfloor HitLocation.X / S \rfloor \\
    CenterY &= \lfloor HitLocation.Y / S \rfloor \\
    CenterZ &= \lfloor HitLocation.Z / S \rfloor
    \end{aligned}
    $$
2.  **Define Offsets:** Create a list of 7 offset vectors representing the center block and its 6 immediate neighbors:
    $$
    \text{Offsets} = \{ (0,0,0), (0,0,-1), (0,0,1), (-1,0,0), (1,0,0), (0,-1,0), (0,1,0) \}
    $$
3.  **Iterate and Validate:** For each offset $(Off.X, Off.Y, Off.Z)$ in Offsets:
    a.  Compute global block coordinates:
        $$
        \begin{aligned}
        GX &= CenterX + Off.X \\
        GY &= CenterY + Off.Y \\
        GZ &= CenterZ + Off.Z
        \end{aligned}
        $$
    b.  **Boundary Check:** If $GZ < 0$ or $GZ \ge ChunkSizeZ$, skip this offset [S1, unofficial].
    c.  **Coordinate Conversion:** Convert global coordinates to chunk and local block coordinates:
        $$
        GlobalToLocal(GX, GY, GZ, CI, LB)
        $$
        [S1, unofficial]
    d.  **Chunk Retrieval:** Retrieve the chunk data from `ChunkCache` using $CI$. If the data is `null` or the block at $LB$ is `Air`, continue to the next offset [S1, unofficial].
    e.  **Distance Calculation:** Calculate the world-space center of the candidate block:
        $$
        BlockCenter = ((GX + 0.5) \cdot S, (GY + 0.5) \cdot S, (GZ + 0.5) \cdot S)
        $$
        Compute the squared Euclidean distance between `HitLocation` and `BlockCenter`:
        $$
        Dist = \| HitLocation - BlockCenter \|^2
        $$
    f.  **Update Best Match:** If $Dist < BestDist$ (initially infinity), update `BestDist = Dist`, `OutChunkIndex = CI`, `OutLocalBlock = LB`, and set `bFound = true` [S1, unofficial].

4.  **Return Result:** Return `bFound` and the coordinates in `OutChunkIndex` and `OutLocalBlock`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def FindSolidBlock(HitLocation, S, ChunkSizeZ, ChunkCache):
    # Step 1: Calculate center grid coordinates
    CenterX = floor(HitLocation.X / S)
    CenterY = floor(HitLocation.Y / S)
    CenterZ = floor(HitLocation.Z / S)

    # Step 2: Define offsets
    Offsets = [
        (0, 0, 0), (0, 0, -1), (0, 0, 1),
        (-1, 0, 0), (1, 0, 0),
        (0, -1, 0), (0, 1, 0)
    ]

    BestDist = infinity
    OutChunkIndex = None
    OutLocalBlock = None
    bFound = False

    # Step 3: Iterate through offsets
    for Off in Offsets:
        OffX, OffY, OffZ = Off

        # Compute global coordinates
        GX = CenterX + OffX
        GY = CenterY + OffY
        GZ = CenterZ + OffZ

        # Boundary check
        if GZ < 0 or GZ >= ChunkSizeZ:
            continue

        # Convert to chunk and local block coordinates
        # Implementation of GlobalToLocal depends on chunk grid layout
        CI, LB = GlobalToLocal(GX, GY, GZ)

        # Retrieve chunk data
        ChunkData = ChunkCache.get(CI)
        if ChunkData is None:
            continue

        # Check if block is solid (not Air)
        if ChunkData[LBA] == Air:
            continue

        # Calculate block center
        BlockCenterX = (GX + 0.5) * S
        BlockCenterY = (GY + 0.5) * S
        BlockCenterZ = (GZ + 0.5) * S
        BlockCenter = (BlockCenterX, BlockCenterY, BlockCenterZ)

        # Calculate squared distance
        Dist = (HitLocation.X - BlockCenterX)**2 + \
               (HitLocation.Y - BlockCenterY)**2 + \
               (HitLocation.Z - BlockCenterZ)**2

        # Update best match
        if Dist < BestDist:
            BestDist = Dist
            OutChunkIndex = CI
            OutLocalBlock = LB
            bFound = True

    return bFound, OutChunkIndex, OutLocalBlock
```

**Edge Cases and Pitfalls:**
-   **Precision:** Floating-point division and flooring must handle negative coordinates correctly if the world origin is not at $(0,0,0)$.
-   **Boundaries:** The $GZ$ check prevents accessing invalid chunk rows. If the world is vertically infinite, this check may need adjustment or removal [S1, unofficial].
-   **Empty Chunks:** The method assumes `ChunkCache` returns `null` for unloaded chunks, which must be handled to avoid null pointer exceptions.
-   **Tie-Breaking:** If multiple blocks have the same squared distance (e.g., hit exactly on an edge), the first one encountered in the offset order is chosen. The order is deterministic but arbitrary.

## Subtopics

### [Voxel Block Placement (Face Normal Determination)](5224-voxel-block-placement-face-normal-determination.md)
This method determines the global coordinates of an empty voxel position adjacent to a solid block that was hit by a raycast, enabling the placement of new blocks or mining operations. It solves the problem of translating a continuous raycast hit location into a discrete voxel grid coordinate while respecting the geometry of the hit face. It differs from its parent, [Voxel Block Interaction (Raycast Hit Detection)](3912-voxel-block-interaction-raycast-hit-detection.md), which only identifies the hit block; this method extends that result to find the specific neighbor slot for interaction.

## Operating characteristics
-   **Time Complexity:** $O(1)$, as it always checks exactly 7 blocks (or fewer if boundaries are hit).
-   **Memory Complexity:** $O(1)$ auxiliary space.
-   **Strengths:** Fast, deterministic, and simple to implement. It leverages the spatial locality of the hit point.
-   **Limitations:** It assumes the hit location is close to the target block. If the raycast hit point is inaccurate or far from the actual block surface, this method may fail to find the correct block. It does not handle complex geometries or non-axis-aligned blocks.
-   **When to Use:** Ideal for simple voxel worlds with axis-aligned blocks where interaction is based on proximity to a known hit point.
-   **When Not to Use:** Not suitable for worlds with non-cubic blocks, complex collision meshes, or when the hit point is unreliable.

## Background
This method is derived from the implementation details of the `VoxelWorld` project by HawkDev15. It is used in the `GetBlockCoords` function to support block removal and placement operations. The approach is common in voxel games where chunk-based storage is used, and interaction is resolved by checking the immediate vicinity of a raycast hit.

## Sources
[S1] HawkDev15/VoxelWorld, https://github.com/HawkDev15/VoxelWorld, fetched 2026-10-28 (unofficial)
