# Voxel Block Interaction (Raycast Hit Detection)
Also called: FindSolidBlock

## Overview
A method to find the solid voxel block closest to a raycast hit location by checking the hit block and its 6 immediate neighbors [S1, unofficial]. It differs from brute-force stepping [3780-voxel-raycasting-for-block-interaction.md] by limiting the search to a constant number of grid cells [G].

## How it works
**Inputs:**
- `HitLocation`: World-space vector where the ray intersects the voxel mesh surface [S1, unofficial].
- `S`: Block size (side length of a voxel), accessed via `AWorldMeshActor::BlockSize` [S1, unofficial].
- `ChunkSizeZ`: Vertical dimension parameter used for boundary checking [S1, unofficial].
- `ChunkCache`: Data structure mapping chunk indices to block data [S1, unofficial].

**Outputs:**
- `OutChunkIndex`: 2D integer vector (`FIntPoint`) representing the chunk coordinates [S1, unofficial].
- `OutLocalBlock`: 3D integer vector (`FIntVector`) representing the local block coordinates within the chunk [S1, unofficial].
- `bFound`: Boolean flag indicating whether a solid block was found [S1, unofficial].

**Data Structures:**
- `FIntPoint`: 2D integer vector for chunk indices [S1, unofficial].
- `FIntVector`: 3D integer vector for local block coordinates [S1, unofficial].
- `ChunkCache`: Map from `FIntPoint` to chunk data, where each chunk stores block types indexed by `FIntVector` [S1, unofficial].

**Steps:**
1. Calculate the integer grid coordinates of the hit location:
   $$ \text{CenterX} = \lfloor \text{HitLocation}.X / S \rfloor $$
   $$ \text{CenterY} = \lfloor \text{HitLocation}.Y / S \rfloor $$
   $$ \text{CenterZ} = \lfloor \text{HitLocation}.Z / S \rfloor $$
   [S1, unofficial]

2. Define the set of offsets to check, including the center and its 6 face-adjacent neighbors:
   $$ \text{Offsets} = \{ (0,0,0), (0,0,-1), (0,0,1), (-1,0,0), (1,0,0), (0,-1,0), (0,1,0) \} $$
   [S1, unofficial]

3. Initialize tracking variables: `BestDist` to a large value, `bFound` to false, and `OutChunkIndex`/`OutLocalBlock` to default values [G].

4. Iterate through each `Off` in `Offsets`:
   a. Compute global coordinates:
      $$ \text{GX} = \text{CenterX} + \text{Off}.X $$
      $$ \text{GY} = \text{CenterY} + \text{Off}.Y $$
      $$ \text{GZ} = \text{CenterZ} + \text{Off}.Z $$
      [S1, unofficial]
   b. Skip if `GZ < 0` or `GZ >= ChunkSizeZ` [S1, unofficial].
   c. Convert global coordinates to chunk index and local block coordinates using `GlobalToLocal(GX, GY, GZ, CI, LB)` [S1, unofficial].
   d. Retrieve chunk data from `ChunkCache` using `CI`. If data is null or the block at `LB` is `Air`, continue to the next offset [S1, unofficial].
   e. Calculate block center:
      $$ \text{BlockCenter} = ((\text{GX} + 0.5) \times S, (\text{GY} + 0.5) \times S, (\text{GZ} + 0.5) \times S) $$
      [S1, unofficial]
   f. Calculate squared distance:
      $$ \text{Dist} = \|\text{HitLocation} - \text{BlockCenter}\|^2 $$
      [S1, unofficial]
   g. If `Dist < BestDist`, update `BestDist = Dist`, `OutChunkIndex = CI`, `OutLocalBlock = LB`, and set `bFound = true` [S1, unofficial].

5. Return `bFound` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def FindSolidBlock(HitLocation, S, ChunkSizeZ, ChunkCache, ChunkSizeX, ChunkSizeY):
    CenterX = int(HitLocation.X // S)
    CenterY = int(HitLocation.Y // S)
    CenterZ = int(HitLocation.Z // S)
    
    Offsets = [
        (0, 0, 0), (0, 0, -1), (0, 0, 1),
        (-1, 0, 0), (1, 0, 0),
        (0, -1, 0), (0, 1, 0)
    ]
    
    BestDist = float('inf')
    bFound = False
    OutChunkIndex = None
    OutLocalBlock = None
    
    for Off in Offsets:
        GX = CenterX + Off[0]
        GY = CenterY + Off[1]
        GZ = CenterZ + Off[2]
        
        if GZ < 0 or GZ >= ChunkSizeZ:
            continue
            
        # Standard chunk coordinate conversion [G]
        CI = (GX // ChunkSizeX, GY // ChunkSizeY)
        LB = (GX % ChunkSizeX, GY % ChunkSizeY, GZ)
        
        ChunkData = ChunkCache.get(CI)
        if ChunkData is None:
            continue
            
        BlockType = ChunkData.GetBlockAt(LB)
        if BlockType == 0:  # 0 represents Air
            continue
            
        BlockCenter = (
            (GX + 0.5) * S,
            (GY + 0.5) * S,
            (GZ + 0.5) * S
        )
        
        Dist = (HitLocation.X - BlockCenter[0])**2 + \
               (HitLocation.Y - BlockCenter[1])**2 + \
               (HitLocation.Z - BlockCenter[2])**2
               
        if Dist < BestDist:
            BestDist = Dist
            OutChunkIndex = CI
            OutLocalBlock = LB
            bFound = True
            
    return bFound, OutChunkIndex, OutLocalBlock
```

## Subtopics

### Voxel Block Placement (Face Normal Determination)
A method to determine the empty voxel position adjacent to a hit solid block, based on which face of the block was hit [S1, unofficial]. It calls `FindSolidBlock` to get the solid block's chunk index and local block coordinates [S1, unofficial]. It calculates the global coordinates of the solid block: `SolidGX = SolidChunk.X * ChunkSizeX + SolidBlock.X`, `SolidGY = SolidChunk.Y * ChunkSizeY + SolidBlock.Y`, `SolidGZ = SolidBlock.Z` [S1, unofficial]. It calculates the solid block center: `SolidCenter = ((SolidGX + 0.5f) * S, (SolidGY + 0.5f) * S, (SolidGZ + 0.5f) * S)` [S1, unofficial]. It calculates the offset vector: `Offset = HitLocation - SolidCenter` [S1, unofficial]. It determines the hit face axis by comparing absolute values of Offset components. If `AbsX` is largest, `PlaceGX += (Offset.X > 0) ? 1 : -1`. If `AbsY` is largest, `PlaceGY += (Offset.Y > 0) ? 1 : -1`. Otherwise, `PlaceGZ += (Offset.Z > 0) ? 1 : -1` [S1, unofficial]. It validates `PlaceGZ` is within `[0, ChunkSizeZ)`. If not, it returns false [S1, unofficial]. It converts `PlaceGX, PlaceGY, PlaceGZ` to chunk index and local block coordinates. It checks if the block at that location is Air. If not, it returns false [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$, checking a constant 7 blocks regardless of world size [G].
- **Memory Complexity:** $O(1)$ auxiliary space [G].
- **Strengths:** Fast execution suitable for real-time interaction [G].
- **Limitations:** Limited to immediate neighbors; may miss blocks if the ray passes through gaps larger than one block width without intersecting a neighbor [G].
- **When to use:** Player interaction (breaking/placing) near the raycast hit surface [G].
- **When not to use:** Long-range selection or when hit location is not surface-adjacent [G].

## Background
Not given in the sources.

## Sources
[S1] HawkDev15/VoxelWorld, https://github.com/HawkDev15/VoxelWorld, fetched 2026-10-28 (unofficial)
