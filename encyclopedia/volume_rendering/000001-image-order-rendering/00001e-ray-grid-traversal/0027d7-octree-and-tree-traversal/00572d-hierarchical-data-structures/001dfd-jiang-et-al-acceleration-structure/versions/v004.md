# Jiang et al. Acceleration Structure
Also called: JRZ*16 method

## Overview
The Jiang et al. acceleration structure [JRZ*16] is a hierarchical data structure designed to efficiently locate active cells for both direct and indirect volume visualization approaches [S1]. It solves the problem of redundant ray marching through inactive (empty or background) regions by pruning the search space [S1]. Unlike its sibling [Hierarchical Spatial Enumeration Optimization](1663-hierarchical-spatial-enumeration-optimization.md) which focuses on spatial enumeration, this method specifically targets hybrid object-order and image-order rendering, making it dependent on the rendering resolution [S1].

## How it works
The method builds upon the shared principles of hierarchical data structures [Parent]. It constructs a hierarchy where internal nodes aggregate voxel properties to enable early ray termination or skipping of empty subtrees [S1, Parent]. The structure supports ray marching through active cells identified by the hierarchy [S1].

**Inputs and Outputs:**
*   **Input:** A scalar volume dataset.
*   **Output:** A hierarchical acceleration structure capable of identifying active cells for ray marching [S1].

**Data Structures:**
Not given in the sources. The structure relies on the general hierarchical aggregation model described in [Parent], but specific field layouts, bit-packing, or node sizes are not detailed in [S1].

**Construction and Traversal:**
1.  **Hierarchy Construction:** The volume is subdivided hierarchically. Internal nodes store aggregate information derived from their children or leaf voxels [Parent].
2.  **Active Cell Identification:** The structure is used to efficiently locate active cells [S1].
3.  **Ray Marching:** Active cells are rendered using ray marching [S1].

**Parameters:**
Not given in the sources.

**Edge Cases and Pitfalls:**
Not given in the sources.

## Subtopics

### IsoBAS Acceleration Structure
The IsoBAS acceleration structure [LCD15] is a related method introduced by Liu et al. for faster identification of active cells with less memory overhead than previous methods [S1]. It is cited in the same context as the Jiang et al. structure as a hybrid approach [S1].

## Operating characteristics
The Jiang et al. method is characterized as a hybrid object-order and image-order approach [S1]. This implies a dependency on the rendering resolution [S1]. Specific time and memory complexity metrics, measured performance data, or hardware scaling details are not provided in the sources [S1].

## Background
The method was proposed by Jiang et al. [JRZ*16] as an acceleration structure for volume rendering [S1]. It is part of a lineage of hybrid approaches that includes [IsoBAS Acceleration Structure](1663-hierarchical-spatial-enumeration-optimization.md) (Liu et al. [LCD15]) and earlier methods [LCD09] [S1]. These methods aim to improve performance by efficiently locating active cells for ray marching [S1].

## Sources
[S1] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://www.uni-weimar.de/en/media/chairs/computer-science-department/vr/research/real-time-rendering/efficient-direct-isosurface-rasterization-of-scalar-volumes/, fetched 2026-11-01
