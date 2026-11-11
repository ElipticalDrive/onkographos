# From-Region Visibility Preprocessing

Also called: Durand et al. Visibility Preprocessing, Wonka et al. Visibility Preprocessing

## Overview
From-region visibility preprocessing is a technique that precomputes which parts of a 3D scene are visible from specific regions of the camera space, allowing the renderer to discard occluded geometry before the frame is drawn. It solves the problem of expensive per-frame visibility testing for large datasets by shifting the computational burden to an offline phase. Unlike sibling methods such as [Block Visibility Culling](5334-block-visibility-culling.md) which operate on voxel blocks, this method typically operates on high-polygon triangle meshes or detailed geometric representations, distinguishing it from voxel-based culling approaches.

## How it works
The method involves constructing a visibility map or occluder hierarchy that associates regions of the viewing volume with subsets of the scene geometry.

**Inputs and Outputs:**
- **Input:** A 3D scene model (typically a triangle mesh) and a set of predefined camera regions or viewpoints.
- **Output:** A data structure mapping each region to a list of visible primitives (triangles) and potentially occluders.

**Data Structures:**
The specific layout is not fully detailed in the sources, but generally involves:
- A region definition (e.g., a bounding box or frustum).
- A list of visible triangles for that region.
- Occluder information used to cull triangles for other regions.

**Steps:**
1. **Region Definition:** Divide the camera space into discrete regions. The sources do not specify the exact partitioning strategy (e.g., grid, octree, or manual placement) [S1, unofficial].
2. **Visibility Computation:** For each region, determine which triangles are visible. This often involves rendering the scene from the region's viewpoint(s) and identifying visible primitives.
3. **Occluder Fusion:** Identify triangles that act as occluders for other regions. These occluders are used to cull triangles in other regions' visibility lists [S1, unofficial].
4. **Preprocessing Storage:** Store the resulting visibility maps for runtime lookup.

**Parameters:**
- **Number of Regions:** Not specified.
- **Preprocessing Time:** Highly dependent on scene complexity. For a model with 8 million triangles, preprocessing took 9 hours [S1, unofficial]. For a model with 6 million triangles, it took 33 hours [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Long Preprocessing Times:** The method can require significant offline computation time, making it less suitable for dynamic scenes or rapid iteration [S1, unofficial].
- **Dataset Limitations:** Early implementations were limited to 2.5D datasets [S1, unofficial].

## Subtopics

### Chhugani et al. Visibility and LOD System
A system that precomputes from-region visibility and levels of detail per region, focusing on image accuracy [S1, unofficial]. It guarantees less than one pixel of screen-space deviation and correct visibility. For a model with 13 million triangles, using a cluster of 16 PCs, the accuracy guarantee costs 128 hours of precomputation [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Preprocessing time is high, scaling with the number of triangles and regions. Reported times range from 9 to 128 hours for models with 6 to 13 million triangles [S1, unofficial].
- **Memory Complexity:** Not specified.
- **Performance:** Enables interactive rendering by reducing the number of primitives processed per frame [S1, unofficial].
- **Strengths:** High image accuracy with correct visibility; suitable for large static datasets [S1, unofficial].
- **Limitations:** Very long preprocessing times; early versions limited to 2.5D datasets [S1, unofficial].

## Background
The concept was introduced by Wonka et al. [152] as a from-region visibility preprocessing algorithm with occluder fusion, which used 2 processors to overlap visibility computation and rendering at runtime [S1, unofficial]. Durand et al. [41] extended this to handle 3D environments, though it required long preprocessing times (33 hours for 6 million triangles) [S1, unofficial]. Later work by Wonka et al. [153] shifted to a from-point approach with less preprocessing but remained limited to 2.5D datasets [S1, unofficial]. Chhugani et al. [20] developed a system focusing on image accuracy and LOD per region [S1, unofficial].

## Sources
[S1] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
