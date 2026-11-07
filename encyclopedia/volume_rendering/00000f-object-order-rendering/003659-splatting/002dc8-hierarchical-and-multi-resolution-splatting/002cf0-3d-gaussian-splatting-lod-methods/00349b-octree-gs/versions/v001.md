# Octree-GS
Also called: Octree Gaussian Splatting
## Overview
Octree-GS is a hierarchical level-of-detail (LOD) method for 3D Gaussian Splatting that builds an octree over the scene, associating each node with a merged Gaussian that summarizes the radiance and opacity of its children [S1, unofficial]. It introduces an LOD strategy to better capture details, building upon the Scaffold-GS representation [S2]. It sets itself apart from sibling methods like CityGaussian by using a strict octree hierarchy for merging rather than voxel-grid scaffolding or clustering [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
- Input: A set of 3D Gaussians $\{G_i\}$ from a base 3DGS reconstruction, and an octree structure.
- Output: A hierarchical representation allowing rendering at varying levels of detail.

**Data Structures:**
- **Octree Node:** Each node contains a pointer to children (if not a leaf) and a merged Gaussian attribute.
- **Merged Gaussian:** A single Gaussian $G_{node}$ that summarizes the radiance and opacity of its children [S1, unofficial]. The exact mathematical formulation for this merge (e.g., weighted average of means, covariances, opacities) is not given in the sources.

**Steps:**
1. **Initialize Scaffold:** Build upon the Scaffold-GS representation, which anchors scene content to a sparse voxel grid [S2].
2. **Build Octree:** Construct an octree over the scene geometry or occupied regions [S1, unofficial].
3. **Merge Gaussians:** For each octree node, compute a merged Gaussian that summarizes the radiance and opacity of its child nodes [S1, unofficial].
4. **LOD Selection:** During rendering, select the appropriate level of detail (node depth) based on view distance or resolution requirements [S2].

**Parameters:**
- Octree depth/LOD levels: Not given in the sources.
- Merge weights: Not given in the sources.

**Edge Cases:**
- Degenerate nodes (empty children): Not specified.
- Precision issues in merging: Not specified.

## Subtopics

### [CityGaussian / VastGaussian / DoGaussian](6143-citygaussian-vastgaussian-dogaussian.md)
Uses voxel-based scaffolding and clustering. Choose when memory efficiency via sparse voxels is prioritized over strict hierarchical merging.

### [LightGaussian](8036-lightgaussian.md)
Focuses on lightweight representation. Choose for real-time performance on constrained hardware.

### [Multi-scale filtering](8244-multi-scale-filtering.md)
Uses filtering for LOD. Choose for smooth transitions between scales.

### [Virtual Memory for 3D Gaussian Splatting](8403-virtual-memory-for-3d-gaussian-splatting.md)
Manages memory via virtualization. Choose for extremely large scenes exceeding VRAM.

### [Gaussian clustering and merging](8436-gaussian-clustering-and-merging.md)
General clustering techniques. Choose when flexible grouping is needed.

### [Iterative Gaussian Synopsis](9289-iterative-gaussian-synopsis.md)
Uses iterative synopsis. Choose for progressive simplification.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Improves memory efficiency compared to base 3DGS by allocating Gaussians only in occupied regions and using hierarchical merging [S1, unofficial].
- **Performance:** Enables real-time rendering for large datasets through adaptive level selection [S1, unofficial].
- **Strengths:** Better detail capture via LOD [S2]; memory efficiency [S1, unofficial].
- **Limitations:** Requires octree construction and merging logic; potential artifacts if merge is not faithful [S2].

## Background
Octree-GS was introduced as an extension of Scaffold-GS [S2]. Scaffold-GS anchors scene content to a sparse voxel grid, using the grid as a scaffold to allocate Gaussians only in occupied regions, which naturally adapts to view distance and improves memory efficiency [S1, unofficial]. Octree-GS builds on this by introducing a level-of-detail strategy to better capture details [S2]. It is part of the broader effort to scale 3D Gaussian Splatting to large scenes [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
