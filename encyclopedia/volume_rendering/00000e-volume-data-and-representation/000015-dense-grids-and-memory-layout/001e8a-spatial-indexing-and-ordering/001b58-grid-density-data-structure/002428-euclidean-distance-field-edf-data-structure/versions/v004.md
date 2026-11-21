# Euclidean Distance Field (EDF) Data Structure

Also called: EuclideanDF, Tiled Euclidean Distance Fields

## Overview
The Euclidean Distance Field (EDF) is a dense grid data structure used in volume rendering to accelerate ray traversal by storing the distance to the nearest occupied voxel for every location in the volume [S1, unofficial]. It solves the problem of slow, voxel-by-ray stepping in dense or complex geometries by allowing rays to "skip" empty space in large increments [S1, unofficial]. It is distinguished from its siblings, such as the [VoxelGrid Data Structure](1216-voxelgrid-data-structure.md) which steps one voxel at a time, and the [EdgeMap Data Layout](1109-edgemap-data-layout.md) which uses boundary representations, by providing a continuous metric of proximity that enables significant traversal speedups at the cost of higher memory bandwidth and pre-processing time [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A binary occupancy grid (voxels are either empty or occupied) and a target resolution scale.
*   **Output:** A tiled EDF grid containing distance values and an occupancy bitmask grid.

**Data Structures:**
The implementation uses a tiled layout to balance memory cost and traversal efficiency.
1.  **Distance Field Grid:** Stores 16-bit unsigned integers. The resolution is scaled down by a factor of 4 (1:4 scale). One distance value represents a $4^3 = 64$ voxel block.
    *   **Layout:** 2 bytes per $4^3$ voxel block.
2.  **Occupancy Bitmask Grid:** Stores a single bit per $1^3$ voxel to indicate exact occupancy within the coarse blocks.
    *   **Layout:** 1 bit per $1^3$ voxel.

**Algorithm Steps:**
1.  **Compute Distance Field:** Calculate the Euclidean distance from every voxel to the nearest occupied voxel. This is typically done using an exact distance transform algorithm (e.g., Felzenswalb) to avoid propagation errors [S1, unofficial].
2.  **Downsample:** Store the computed distances in the coarse grid at 1:4 resolution.
3.  **Generate Bitmasks:** Create a bitmask for every $4^3$ block, where the $i$-th bit is set if the $i$-th voxel in that block is occupied.
4.  **Ray Traversal:**
    *   Cast a ray through the volume.
    *   Query the EDF grid to get the distance $d$ to the nearest occupied voxel.
    *   If $d > 0$, advance the ray position by $d$.
    *   If $d = 0$, the ray is at or near an occupied voxel. Query the occupancy bitmask for the current $4^3$ block.
    *   Use the bitmask to determine the exact next occupied voxel or to refine the traversal step if the geometry is complex (e.g., grazing rays in dense foliage) [S1, unofficial].

**Parameters:**
*   **Scale Factor:** 4 (1:4 resolution). This was chosen to reduce memory costs without significantly harming traversal efficiency [S1, unofficial].
*   **Tile Size:** $256^3$ voxels. The EDF is tiled at this resolution for memory management [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Grazing Rays:** Dense areas with fine geometry (like foliage) cause grazing rays to advance slowly even with EDFs, as the distance values may be small frequently [S1, unofficial].
*   **Pre-processing Cost:** Computing the EDF has a significant cost, reported as 10x more GPU time than alternative methods like XBrickMap [S1, unofficial].
*   **Memory Bandwidth:** The method incurs high memory bandwidth costs, which becomes problematic for incoherent rays that lose coherence as they move away from the origin [S1, unofficial].
*   **Traversal Length:** The traversal is not considerably shorter than methods relying on space partitioning (only ~8% shorter), limiting the overall performance gain [S1, unofficial].

## Subtopics

### [Felzenswalb Distance Transform](4676-felzenswalb-distance-transform.md)
The Felzenswalb distance transform is an algorithm for computing the exact Euclidean distance field (EDF) from a binary volume in three passes, eliminating the propagation and numerical errors inherent in approximate methods. It is distinguished from its nearest alternative, the Satherley and Jones method, by its exactness, parallelizability, and significantly higher computational efficiency, though it requires greater memory consumption.

### Manhattan Distance Field (ManhattanDF)
**Definition:** Space skipping traversal using tiled $256^3$ Manhattan distance fields at 1:4 resolution, combined with $4^3$ occupancy bitmasks [S1, unofficial].
**Layout:** Memory cost is 1 byte per $4^3$ voxel plus 1 bit per $1^3$ voxel [S1, unofficial].

### Weighted Euclidean Distance Field
**Definition:** A geometric measure of spatial location used as a diversifying information attribute in Multi-Dimensional Transfer Function (MDTF) design, allowing users to set seed points or define body parts [S2, unofficial].
**Formula:**
$$ \rho(\mathbf{x}) = \left( \sum_{i=1}^3 k_i (x_i - c_i)^2 \right)^{1/2} $$
where $\mathbf{x} = (x_1, x_2, x_3)$ is the voxel position, $(c_1, c_2, c_3)$ is the reference point, and $(k_1, k_2, k_3)$ are optional dimension-specific weighting factors [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Pre-processing is expensive, taking approximately 10x more GPU time than XBrickMap [S1, unofficial]. Traversal speedup is modest, reducing traversal length by only ~8% compared to space-partitioning methods [S1, unofficial].
*   **Memory Complexity:** 2 bytes per $4^3$ voxel for the distance field plus 1 bit per $1^3$ voxel for occupancy masks [S1, unofficial].
*   **Measured Performance:** In dense geometries (foliage), EDFs perform worse than space-partitioning methods. However, directional variants (OctantDF) can lower iterations by ~20-25% and increase throughput for incoherent rays by ~30-50% [S1, unofficial].
*   **Strengths:** Reduces the number of ray steps in empty space; exact distance transforms avoid numerical errors.
*   **Limitations:** High memory bandwidth usage; significant pre-processing cost; limited traversal length reduction compared to spatial partitioning; poor performance on grazing rays in dense volumes [S1, unofficial].
*   **When to use:** When memory bandwidth is not the primary bottleneck and pre-processing time is acceptable; for volumes with large empty regions.
*   **When not to use:** For incoherent rays in dense, fine-geometry volumes where space-partitioning methods offer better performance; when memory bandwidth is severely constrained.

## Background
The Euclidean Distance Field data structure was developed in the context of voxel-based ray tracing, specifically within the VoxelRT project [S1, unofficial]. It was designed to address the performance limitations of naive voxel traversal in dense volumes. The specific tiled implementation with 1:4 resolution and occupancy masks was chosen to balance memory costs and traversal efficiency [S1, unofficial]. The concept of using spatial location as a diversifying attribute in transfer functions, including weighted Euclidean distances, was introduced in the context of Multi-Dimensional Transfer Function (MDTF) design for medical imaging [S2, unofficial].

## Sources
[S1] dubiousconst282/VoxelRT, https://github.com/dubiousconst282/VoxelRT, fetched 2026-10-24 (unofficial)
[S2] Multi-Dimensional Transfer Function Design Using Sorted Histograms, https://doi.org/10.2312/vg/vg06/001-008, fetched 2026-10-26 (unofficial)
