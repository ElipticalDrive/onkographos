# Sparse AABB Collision Test

Also called: TestAabb

## Overview
The Sparse AABB Collision Test determines whether an axis-aligned bounding box (AABB) intersects any occupied voxels within a sparse volume representation [S1, unofficial]. It solves the problem of discrete collision detection for entities or objects moving through a voxel world by sampling the volume defined by the AABB [S1, unofficial]. It differs from its sibling [Point-in-Box Inclusion Test](1400-point-in-box-inclusion-test.md) by classifying samples into specific material types (solid, liquid, unknown) rather than a simple binary occupancy, and from its parent [Voxel Occupancy Test](1281-voxel-occupancy-test.md) by supporting configurable blocking rules for liquids and tracking the first blocking sample [S1, unofficial].

## How it works
**Inputs:**
- `aabb`: A `SparseCollisionAabb` structure defining the test volume with `min` and `max` float coordinates [S1, unofficial].
- `liquidsBlock`: A boolean flag indicating if liquid voxels count as blocking collisions [S1, unofficial].

**Outputs:**
- A result structure containing:
  - `blocked`: Boolean indicating if any blocking voxel was found [S1, unofficial].
  - `solidVoxels`, `liquidVoxels`, `unknownVoxels`, `sampledVoxels`: Integer counters for each sample type [S1, unofficial].
  - `hasLiquid`, `hasUnknown`: Boolean flags indicating presence of these types [S1, unofficial].
  - `firstBlockingSample`: The `CollisionSample` data of the first blocking voxel encountered [S1, unofficial].
  - `firstBlockingX`, `firstBlockingY`, `firstBlockingZ`: Integer voxel coordinates of the first blocking voxel [S1, unofficial].

**Data Structures:**
- `SparseCollisionAabb`: Contains `min` and `max` vectors [S1, unofficial].
- `CollisionSample`: Contains `status` (enum: `KnownAir`, `KnownSolid`, `KnownLiquid`, `UnknownBlocked`) and metadata [S1, unofficial].
- `colHeight`, `colRelief`: Precomputed arrays indexed by XZ footprint column, storing terrain height and relief values to cache noise evaluation [S1, unofficial].

**Steps:**
1. **Validate Bounds:** Check if `aabb.min` or `aabb.max` contains non-finite values (NaN or Infinity). If so, return immediately with `UnknownBlockedVolume` status [S1, unofficial].
2. **Voxel Range Conversion:** Convert the AABB's float coordinates to integer voxel indices using `TryBuildVoxelRange`. This function handles float precision at boundaries using `nextafter` for the max-exclusive limit [S1, unofficial]. Let the resulting range be `[minX, maxX]`, `[minY, maxY]`, `[minZ, maxZ]` [S1, unofficial].
3. **Sample Limit Check:** Calculate the total voxel count: $(maxX - minX + 1) \times (maxY - minY + 1) \times (maxZ - minZ + 1)$. If this count exceeds `kMaxCollisionVolumeSamples` (32768), return immediately with `UnknownBlockedVolume` status [S1, unofficial].
4. **Precompute Column Data:** Iterate over the XZ footprint of the AABB [S1, unofficial]. For each column $(x, z)$, compute the terrain height and relief using the volume's noise function [S1, unofficial]. Store these in `colHeight` and `colRelief` arrays indexed by the column offset [S1, unofficial]. This collapses the noise evaluation cost from $O(voxels)$ to $O(columns)$ [S1, unofficial].
5. **Iterate Voxels:** Loop through the voxel grid in order: $Z$ from `minZ` to `maxZ`, $Y$ from `minY` to `maxY`, $X$ from `minX` to `maxX` [S1, unofficial].
6. **Sample Voxel:** For each voxel $(x, y, z)$, calculate the column index and call `SampleWithColumn(x, y, z, colHeight[idx], colRelief[idx])` to get the `CollisionSample` [S1, unofficial].
7. **Classify and Count:** Increment `sampledVoxels` [S1, unofficial]. Based on `sample.status`: `KnownAir` does nothing; `KnownSolid` increments `solidVoxels` and sets `sampleBlocks = true`; `KnownLiquid` increments `liquidVoxels`, sets `hasLiquid = true`, and sets `sampleBlocks = liquidsBlock`; `UnknownBlocked` (or default) increments `unknownVoxels`, sets `hasUnknown = true`, and sets `sampleBlocks = true` [S1, unofficial].
8. **Track First Block:** If `sampleBlocks` is true and `result.blocked` is false, set `result.blocked = true`, record `result.firstBlockingSample`, and store $(x, y, z)$ in `result.firstBlockingX/Y/Z` [S1, unofficial].
9. **Return:** Return the populated result structure [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

# Types and constants defined for self-containment
class SparseCollisionAabb:
    def __init__(self, min, max): self.min, self.max = min, max
class CollisionSample:
    def __init__(self, status): self.status = status
class CollisionResult:
    def __init__(self):
        self.blocked = False
        self.solidVoxels = 0
        self.liquidVoxels = 0
        self.unknownVoxels = 0
        self.sampledVoxels = 0
        self.hasLiquid = False
        self.hasUnknown = False
        self.firstBlockingSample = None
        self.firstBlockingX = 0
        self.firstBlockingY = 0
        self.firstBlockingZ = 0

KnownAir = 0
KnownSolid = 1
KnownLiquid = 2
UnknownBlocked = 3
UnknownBlockedVolume = "UnknownBlockedVolume"

def sparse_aabb_collision_test(aabb: SparseCollisionAabb, liquidsBlock: bool) -> CollisionResult:
    # Step 1: Validate bounds
    if not all(math.isfinite(v) for v in aabb.min) or not all(math.isfinite(v) for v in aabb.max):
        return CollisionResult() # Returns UnknownBlockedVolume status

    # Step 2: Convert to voxel indices
    # TryBuildVoxelRange converts float coordinates to integer voxel indices.
    # It uses nextafter for the max-exclusive boundary to handle float precision.
    minX = int(math.floor(aabb.min.x))
    maxX = int(math.ceil(aabb.max.x)) - 1
    minY = int(math.floor(aabb.min.y))
    maxY = int(math.ceil(aabb.max.y)) - 1
    minZ = int(math.floor(aabb.min.z))
    maxZ = int(math.ceil(aabb.max.z)) - 1

    # Step 3: Check sample limit
    spanX = maxX - minX + 1
    spanY = maxY - minY + 1
    spanZ = maxZ - minZ + 1
    total_voxels = spanX * spanY * spanZ

    if total_voxels > 32768:
        return CollisionResult()

    # Step 4: Precompute column data
    colHeight = [0.0] * (spanX * spanZ)
    colRelief = [0.0] * (spanX * spanZ)
    # Not given in the sources: terrain height and relief computation
    for z in range(minZ, maxZ + 1):
        for x in range(minX, maxX + 1):
            idx = (x - minX) + (z - minZ) * spanX
            # colHeight[idx], colRelief[idx] = compute_terrain_noise(x, z)

    # Initialize result
    result = CollisionResult()

    # Step 5-8: Iterate and sample
    for z in range(minZ, maxZ + 1):
        for y in range(minY, maxY + 1):
            for x in range(minX, maxX + 1):
                idx = (x - minX) + (z - minZ) * spanX
                # Not given in the sources: actual sampling logic using cached height/relief
                sample = CollisionSample(KnownAir)

                result.sampledVoxels += 1

                sampleBlocks = False
                if sample.status == KnownAir:
                    pass
                elif sample.status == KnownSolid:
                    result.solidVoxels += 1
                    sampleBlocks = True
                elif sample.status == KnownLiquid:
                    result.liquidVoxels += 1
                    result.hasLiquid = True
                    sampleBlocks = liquidsBlock
                else: # UnknownBlocked or default
                    result.unknownVoxels += 1
                    result.hasUnknown = True
                    sampleBlocks = True

                if sampleBlocks and not result.blocked:
                    result.blocked = True
                    result.firstBlockingSample = sample
                    result.firstBlockingX = x
                    result.firstBlockingY = y
                    result.firstBlockingZ = z

    return result
```

**Parameters:**
- `kMaxCollisionVolumeSamples`: Constant `uint64_t` value of 32768. Maximum allowed voxel count before early exit [S1, unofficial].

**Edge Cases and Pitfalls:**
- Non-finite AABB bounds trigger an immediate return with `UnknownBlockedVolume` [S1, unofficial].
- Exceeding the sample limit (32768 voxels) triggers an immediate return with `UnknownBlockedVolume` [S1, unofficial].
- `TryBuildVoxelRange` uses `nextafter` for the max-exclusive boundary to handle floating-point precision errors at voxel edges [S1, unofficial].

## Subtopics

### [Sparse AABB Sweep Test](5413-sparse-aabb-sweep-test.md)
The Sparse AABB Sweep Test is a voxel occupancy test that determines whether an axis-aligned bounding box (AABB) moving along a displacement vector intersects any occupied voxels in a sparse volume. It approximates continuous collision detection by sampling the displacement path at discrete intervals. It differs from its sibling [Support Below Search](5414-support-below-search.md) by testing the entire swept volume rather than just the base support, and from its parent [Sparse AABB Collision Test](3047-sparse-aabb-collision-test.md) by handling non-zero displacement vectors through temporal sampling.

### [Support Below Search](5414-support-below-search.md)
Support Below Search determines if a voxel position has structural support directly beneath it within a specified vertical drop distance. It solves the problem of identifying valid standing positions for entities or objects in a voxel world by finding the first non-empty voxel (solid, liquid, or blocked unknown) below a given footprint. It differs from [Sparse AABB Sweep Test](5413-sparse-aabb-sweep-test.md) by iterating through a defined volume to find the *first* support rather than sweeping for *any* collision along a path, and builds on [Sparse AABB Collision Test](3047-sparse-aabb-collision-test.md) by utilizing its collision sample semantics [G].

### Voxel-Primitive Intersection Culling
A technique to cull unwanted voxels generated by conservative rasterization edge expansion [S2, unofficial]. It checks for intersections between the primitive's plane and the voxels in the depth direction within a single pixel [S2, unofficial]. The method splits workload between the Geometry Shader (GS) and Pixel Shader (PS) [S2, unofficial]. In the GS, the primitive's AABB is calculated from vertex min/max coordinates [S2, unofficial]. In the PS, the voxel's AABB is defined by the grid [S2, unofficial]. An AABB test discards voxels that do not intersect the primitive's AABB [S2, unofficial]. If they intersect, an Edge-Voxel Condition Test is performed in each of the three axis-aligned planes (XY, YZ, ZX) [S2, unofficial]. For each edge, the signed distance from the edge to the voxel's vertices is calculated [S2, unofficial]. The maximum signed distance across all voxel vertices is used [S2, unofficial]. If all calculated signed distances are positive, the test passes, and the voxel is covered [S2, unofficial]. The intersection target is the entire box of the voxel [S2, unofficial].

### VoxelBoxMover Collision System
A Minecraft-like collision system provided by the `VoxelBoxMover` class, used as an alternative to mesh-based collisions (Trimesh) which are slow to build and process [S3]. It is faster, not affected by tunnelling, but more limited [S3]. The system initializes with a character bounding box (AABB) and terrain node [S3]. In the physics process, it calculates a motion vector from velocity and delta time, then calls `box_mover.get_motion()` with the current translation, motion vector, character box, and terrain to apply collision [S3]. The global translation and velocity are updated based on the returned motion [S3]. If using `VoxelMesherBlocky`, collision is determined by a list of AABBs in `VoxelBlockyModel` resources [S3]. If the list is empty, the voxel has no collisions [S3]. Collisions can be filtered using `VoxelBoxMover.collision_mask` matched against `VoxelBlockyModel.collision_mask` [S3]. This technique mainly works with `VoxelMesherBlocky`, which gets collidable block info from `VoxelBlockyLibrary` [S3]. It may have limited support in other meshers [S3].

## Operating characteristics
- **Time Complexity:** $O(columns)$ for cache initialization + $O(voxels)$ for sampling. The method collapses terrain-noise evaluation cost from $O(voxels)$ to $O(columns)$ by caching per-column height/relief [S1, unofficial].
- **Memory Complexity:** Not given in the sources.
- **Performance:** The sample limit of 32768 voxels ensures bounded execution time, preventing performance spikes from large AABBs [S1, unofficial].
- **Strengths:** Efficient noise caching; configurable blocking rules for liquids; handles sparse volumes effectively.
- **Limitations:** Limited to AABB shapes; discrete sampling may miss thin features if not carefully tuned (though this specific test is discrete, not swept); limited support for non-blocky meshers in related systems [S3].

## Background
The Sparse AABB Collision Test is part of the VENPOD voxel physics engine implementation [S1, unofficial]. It is designed for high-performance voxel simulation, targeting 100M+ active voxels at 60 FPS [S1, unofficial]. The method leverages precomputed column data to optimize noise evaluation [S1, unofficial].

## Sources
[S1] asajid2-cell/voxelrender, https://github.com/asajid2-cell/voxelrender, fetched 2026-10-27 (unofficial)
[S2] https://developer.nvidia.com/content/basics-gpu-voxelization, https://developer.nvidia.com/content/basics-gpu-voxelization, fetched 2026-10-28 (unofficial)
[S3] Blocky terrains - Voxel Tools documentation, https://voxel-tools.readthedocs.io/en/latest/blocky_terrain/, fetched 2026-10-27
