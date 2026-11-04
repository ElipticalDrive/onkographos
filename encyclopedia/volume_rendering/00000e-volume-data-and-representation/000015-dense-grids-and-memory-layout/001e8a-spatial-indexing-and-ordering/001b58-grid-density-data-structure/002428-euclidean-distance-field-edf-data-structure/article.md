# Euclidean Distance Field (EDF) Data Structure

Also called: EuclideanDF, Tiled Euclidean Distance Fields

## Overview
The Euclidean Distance Field (EDF) is a spatial indexing structure used for space-skipping ray traversal in dense voxel volumes. It accelerates rendering by allowing rays to jump over empty space based on precomputed distance values, reducing the number of intersection tests. It differs from sibling structures like [EdgeMap Data Layout](1109-edgemap-data-layout.md) by storing scalar distance magnitudes rather than boundary flags, and from [Gradient Computation via Finite Differences](1077-gradient-computation-via-finite-differences.md) by providing traversal acceleration rather than surface normal estimation.

## How it works
The EDF structure combines a coarse-resolution distance field with a fine-resolution occupancy mask to balance memory bandwidth and traversal efficiency.

**Inputs and Outputs**
*   **Input:** A binary volume (occupied vs. empty voxels).
*   **Output:** A tiled data structure containing distance values and occupancy bitmasks.

**Data Structures and Layout**
The structure uses a 1:4 resolution scale, meaning one distance value represents a $4^3 = 64$ voxel block.
*   **Distance Field:** Stores 2-byte (16-bit) unsigned integers per $4^3$ voxel block [S1, unofficial].
*   **Occupancy Mask:** Stores 1-bit flags per $1^3$ voxel, packed into 64-bit integers (8 bytes) to represent the $4^3$ block [S1, unofficial].
*   **Total Memory Cost:** 2 bytes per $4^3$ voxel (distance) + 1 bit per $1^3$ voxel (occupancy) [S1, unofficial].

**Algorithm: Traversal**
1.  **Initialization:** Load the ray origin and direction. Determine the starting tile in the EDF grid.
2.  **Tile Query:** Access the EDF tile corresponding to the current ray position.
3.  **Distance Check:** Retrieve the precomputed Euclidean distance value $D$ for the current tile.
4.  **Skip Logic:**
    *   If the ray is in empty space, advance the ray position by $D$ units along the ray direction.
    *   If the distance value is zero or the ray enters a region with fine geometry, fall back to fine-grained traversal (e.g., DDA) or check the occupancy mask.
5.  **Occupancy Check:** If the ray is near a boundary, query the 64-bit occupancy bitmask for the specific $4^3$ block to determine if any voxels are occupied [S1, unofficial].
6.  **Iteration:** Repeat until the ray exits the volume or hits an occupied voxel.

**Parameters**
*   **Resolution Scale:** 1:4 (one distance value per 64 voxels) is the standard implementation [S1, unofficial].
*   **Bitmask Width:** 64-bit integers are used to pack the $4^3$ occupancy bits [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Grazing Rays:** Dense areas with fine geometry (e.g., foliage) cause grazing rays to advance slowly, leading to many small steps despite the EDF [S1, unofficial].
*   **Preprocessing Cost:** Generating the EDF has a significant computational cost, estimated at 10x more GPU time than alternative structures like XBrickMap [S1, unofficial].
*   **Traversal Overhead:** Ascending and descending the tree structure to advance positions via DDA can introduce extra iterations, potentially negating benefits for smooth surfaces [S1, unofficial].

## Subtopics

### Manhattan Distance Field (ManhattanDF)
A variant of the EDF that uses Manhattan distance (L1 norm) instead of Euclidean distance (L2 norm). It uses the same tiled 256³ structure at 1:4 resolution with $4^3$ occupancy bitmasks. Its memory cost is lower: 1 byte per $4^3$ voxel plus 1 bit per $1^3$ voxel [S1, unofficial]. It is chosen when memory bandwidth is the primary constraint and the approximation error of Manhattan distance is acceptable for the scene geometry.

### Felzenswalb Distance Transform
A method to calculate the exact Euclidean distance field from a binary volume in only 3 passes, without propagation or numerical errors [S2, unofficial]. It is used to generate the scalar distance field required for isosurface extraction. Compared to the Satherley and Jones method, it is exact, parallelizable, and significantly faster (270 seconds serial, 47 seconds parallel on 8 threads vs. 722 seconds for Satherley and Jones) [S2, unofficial]. The trade-off is higher memory consumption during calculation [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Traversal is $O(N)$ where $N$ is the number of skipped steps, but preprocessing is expensive (10x GPU time vs. XBrickMap) [S1, unofficial].
*   **Memory Complexity:** 2 bytes per $4^3$ voxel for distance + 1 bit per $1^3$ voxel for occupancy [S1, unofficial].
*   **Performance:** Space skipping is not considerably shorter than space partitioning methods (~8% improvement) [S1, unofficial]. Throughput for incoherent rays is comparable to other methods, possibly due to low memory bandwidth constraints [S1, unofficial].
*   **Strengths:** Reduces memory bandwidth pressure by skipping empty space; simple data layout.
*   **Limitations:** Poor performance on dense, fine geometry (foliage); high preprocessing cost; limited speedup over partitioning methods [S1, unofficial].
*   **When to use:** Scenes with large empty volumes and coarse geometry where preprocessing time is acceptable.
*   **When not to use:** Dense, high-frequency geometry or scenes requiring very fast preprocessing.

## Background
The EDF structure was developed as part of the VoxelRT project, an unofficial voxel ray tracer [S1, unofficial]. It builds on the concept of tiled distance fields to optimize ray traversal in dense volumes. The Felzenswalb distance transform method, often used to generate the underlying data for EDFs, was introduced by Felzenswalb et al. in 2012 [S2, unofficial].

## Sources
[S1] dubiousconst282/VoxelRT, https://github.com/dubiousconst282/VoxelRT, fetched 2026-10-24 (unofficial)
[S2] EXTRAÇÃO DE ISOSUPERFÍCIES DE DOMOS DE SAL EM VOLUMES BINÁRIOS MASSIVOS, https://doi.org/10.17771/pucrio.acad.51290, fetched 2026-10-26 (unofficial)
