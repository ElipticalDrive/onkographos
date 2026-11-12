# Jiang et al. Acceleration Structure
Also called: JRZ*16 method

## Overview
The Jiang et al. acceleration structure [JRZ*16] is a hierarchical data structure designed to efficiently locate active cells for both direct and indirect volume visualization approaches [S1]. It solves the problem of redundant sampling in sparse volumes by enabling ray marching to skip inactive regions [S1]. What sets it apart from its siblings, such as the IsoBAS structure [LCD15], is its unified support for both direct and indirect visualization techniques, whereas IsoBAS focuses primarily on faster identification with lower memory overhead [S1, S2]. Like other methods in this family, it is a hybrid object-order and image-order approach, meaning its performance is dependent on the rendering resolution [S1].

## How it works
The method constructs a spatial subdivision hierarchy to prune empty or inactive voxels during ray traversal [S1].

**Inputs:** A scalar volume dataset; rendering resolution.
**Outputs:** A hierarchical acceleration structure (tree) and a ray-marched image.

**Data Structures:**
The structure is a tree where internal nodes store aggregate information derived from their children or leaf voxels [Parent](4133-hierarchical-data-structures.md).
*   **Leaf Nodes:** Encode voxel properties (e.g., occupancy, scalar values) [Parent](4133-hierarchical-data-structures.md).
*   **Internal Nodes:** Aggregate properties from children, such as binary occupancy flags or min/max scalar ranges [Parent](4133-hierarchical-data-structures.md).

**Construction:**
1.  The hierarchy is constructed bottom-up [Parent](4133-hierarchical-data-structures.md).
2.  Leaf nodes are populated with volume data [Parent](4133-hierarchical-data-structures.md).
3.  Internal nodes are computed by aggregating child data [Parent](4133-hierarchical-data-structures.md).
4.  The depth is determined by data dimensions, often requiring padding to powers of two or specific size constraints to maintain regular subdivision [Parent](4133-hierarchical-data-structures.md).

**Traversal:**
1.  For each pixel, cast a ray into the volume [S1].
2.  Traverse the hierarchy starting from the root [Parent](4133-hierarchical-data-structures.md).
3.  At each node, check the aggregate information (e.g., occupancy flag) [Parent](4133-hierarchical-data-structures.md).
4.  If the node indicates an empty or inactive region (e.g., occupancy flag is 0, or min/max bounds are outside the transfer function's opacity range), skip the entire subtree [Parent](4133-hierarchical-data-structures.md).
5.  If the node is active, traverse its children or perform ray marching at the leaf level [S1].
6.  Composite the results using front-to-back rendering [S1].

**Parameters:**
*   Rendering resolution: Affects performance as the method is resolution-dependent [S1].
*   Not given in the sources.

**Edge Cases and Pitfalls:**
*   Precision issues may arise if aggregate bounds are not tightly packed, leading to false positives in pruning [Parent](4133-hierarchical-data-structures.md).
*   Degenerate input (e.g., fully empty volume) should be handled by early termination at the root [Parent](4133-hierarchical-data-structures.md).

## Subtopics

### IsoBAS Acceleration Structure
An acceleration structure introduced by Liu et al. [LCD15] for faster identification of active cells with less memory overhead than previous methods [S1]. It is chosen when memory efficiency is prioritized over the unified direct/indirect support of the Jiang et al. method [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Performance is dependent on rendering resolution [S1].
*   **Strengths:** Efficiently locates active cells for both direct and indirect visualization approaches [S1].
*   **Limitations:** Hybrid object-order and image-order approach, meaning it is dependent on rendering resolution [S1].
*   **When to use:** When both direct and indirect visualization are required and the volume is sparse [S1].
*   **When not:** When memory overhead is the primary constraint (consider IsoBAS [LCD15]) or when resolution independence is required [S1].

## Background
The Jiang et al. acceleration structure was proposed by Jiang et al. [JRZ*16] as an efficient method for locating active cells in volume rendering [S1]. It is part of a lineage of hybrid object-order and image-order approaches that include earlier methods like those by Liu et al. [LCD09, LCD15] [S1]. The method builds on the principles of hierarchical data structures, where internal nodes aggregate information to enable pruning during ray traversal [Parent](4133-hierarchical-data-structures.md).

## Sources
[S1] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://www.uni-weimar.de/en/media/chairs/computer-science-department/vr/research/real-time-rendering/efficient-direct-isosurface-rasterization-of-scalar-volumes/, fetched 2026-11-01
