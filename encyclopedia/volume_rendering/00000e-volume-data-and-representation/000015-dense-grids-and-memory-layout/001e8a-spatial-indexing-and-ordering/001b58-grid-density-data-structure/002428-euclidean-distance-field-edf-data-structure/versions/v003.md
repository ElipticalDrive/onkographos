# Euclidean Distance Field (EDF) Data Structure
Also called: EuclideanDF, Tiled Euclidean Distance Fields

## Overview
The Euclidean Distance Field (EDF) is a dense grid-based spatial indexing structure used to accelerate ray traversal in volume rendering by enabling large, coherent steps through empty space. It differs from its siblings, such as the [EdgeMap Data Layout](1109-edgemap-data-layout.md) and [VoxelGrid Data Structure](1216-voxelgrid-data-structure.md), by storing precomputed distance values to the nearest occupied voxel rather than raw occupancy or gradient data, allowing traversal to skip directly to the next potential intersection.

## How it works
**Inputs and Outputs:**
- Input: A binary volume (occupied vs. empty voxels).
- Output: A tiled EDF grid and an occupancy bitmask grid.

**Data Structures:**
The structure uses a tiled layout with a resolution reduction factor of 1:4.
1.  **Distance Field:** Stores 16-bit unsigned integers. One value covers a $4^3$ block of voxels (64 voxels).
2.  **Occupancy Mask:** Stores 1 bit per $1^3$ voxel. This acts as a fine-grained occupancy check.

**Memory Layout:**
- Distance Field: 2 bytes per $4^3$ voxel block.
- Occupancy Mask: 1 bit per $1^3$ voxel.
- Total overhead per $4^3$ block: 2 bytes + 8 bits (1 byte).

**Algorithm Steps:**
1.  **Preprocessing:** Compute the Euclidean distance from every voxel to the nearest occupied voxel. Store these values in the Distance Field at 1:4 resolution. Generate the Occupancy Mask for all voxels [S1, unofficial].
2.  **Ray Initialization:** Start ray traversal at the origin.
3.  **Coarse Step:** Query the Distance Field at the current ray position. Let $d$ be the retrieved distance value. Advance the ray position by $d$ units along the ray direction.
4.  **Fine Check:** At the new position, query the Occupancy Mask.
    - If the voxel is empty, repeat Step 3.
    - If the voxel is occupied, the ray has hit a surface or volume boundary. Proceed to shading or further traversal logic.
5.  **Boundary Handling:** If the ray exits the volume bounds, terminate traversal.

**Parameters:**
- Resolution Scale: 1:4 (one distance value per 64 voxels) [S1, unofficial].
- Distance Type: Euclidean [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Zero Distance:** If the sampled distance value is zero, the ray is currently inside or on the boundary of an occupied region. The algorithm must query the occupancy mask immediately to determine if it is inside a solid or on a surface [S1, unofficial].
- **Precision:** Using 16-bit integers for distance may introduce quantization errors in large volumes.
- **Grazing Rays:** Dense geometry causes grazing rays to advance slowly, as the distance to the next occupied voxel is small [S1, unofficial].

## Subtopics

### [Felzenswalb Distance Transform](4676-felzenswalb-distance-transform.md)
The Felzenswalb distance transform is an algorithm for computing the exact Euclidean distance field (EDF) from a binary volume in three passes, eliminating the propagation and numerical errors inherent in approximate methods. It is distinguished from its nearest alternative, the Satherley and Jones method, by its exactness, parallelizability, and significantly higher computational efficiency, though it requires greater memory consumption.

### Manhattan Distance Field (ManhattanDF)
Space skipping traversal using tiled 256³ Manhattan distance fields at 1:4 resolution, combined with 4³ occupancy bitmasks. Memory cost is 1 byte per 4³ voxel plus 1 bit per 1³ voxel [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- Memory: 2 bytes per $4^3$ voxel for EDF + 1 bit per voxel for occupancy [S1, unofficial].
- Preprocessing: Significant cost; approximately 10x more GPU time than XBrickMap [S1, unofficial].

**Performance:**
- Traversal Speed: Space skipping is not considerably shorter than space partitioning methods (~8% improvement) [S1, unofficial].
- Throughput: For incoherent rays, throughput is comparable to other methods, potentially due to low memory bandwidth constraints [S1, unofficial].
- Anisotropic Variants: OctantDF (using 8 separate Manhattan fields) lowers iterations by ~20-25% and increases throughput for incoherent rays by ~30-50% [S1, unofficial].

**Strengths and Limitations:**
- Strengths: Simple traversal logic; effective for sparse volumes.
- Limitations: High preprocessing cost; poor performance on dense geometry (foliage) due to slow ray advancement; overall performance is much worse than space-partitioning methods [S1, unofficial].

## Background
The EDF data structure is used in voxel-based ray traversal to optimize empty space skipping. It builds on the concept of distance transforms, where the Felzenswalb algorithm provides an efficient exact computation method [S1, unofficial]. The specific tiled 1:4 resolution implementation with occupancy masks is described in the VoxelRT project [S1, unofficial].

## Sources
[S1] dubiousconst282/VoxelRT, https://github.com/dubiousconst282/VoxelRT, fetched 2026-10-24 (unofficial)
