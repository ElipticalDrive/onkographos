# Decoupling Resolution Levels from Spatial Subdivision


## Overview
This method decouples the spatial subdivision granularity (leaf-node size) of the residency octree from the resolution levels of the bricked volume hierarchy [S1, unofficial]. It solves the limitation where empty-space skipping efficiency is constrained by the coarse bricking granularity, allowing for finer traversal precision [S1, unofficial]. Unlike coupled approaches where tree structure mirrors data resolution, this technique enables independent optimization of rendering quality and memory scalability .

## How it works
**Inputs:**
- Bricked volume dataset with fixed brick size $B$.
- Residency octree structure.
- Target leaf-node size $L$ for the octree.

**Outputs:**
- Updated residency octree with leaf nodes sized independently of brick dimensions.

**Data Structures:**
- **Brick:** Fixed size $B = 32^3$ voxels [S1, unofficial].
- **Octree Leaf Node:** Variable size $L$, approximating $32^3$, $16^3$, $8^3$, or anisotropic $8 \times 8 \times 4$ [S1, unofficial].

**Steps:**
1. **Initialize Brick Hierarchy:** Construct the bricked volume hierarchy with a fixed brick size of $32^3$ voxels [S1, unofficial].
2. **Configure Octree Granularity:** Select an approximate leaf-node size $L$ for the residency octree that differs from the brick size $B$ [S1, unofficial].
   - Typical values for $L$: $32^3$, $16^3$, $8^3$ [S1, unofficial].
   - Anisotropic value: $8 \times 8 \times 4$ [S1, unofficial].
3. **Map Bricks to Octree Nodes:** Traverse the residency octree. When a node corresponds to a region covered by one or more bricks, update the node's residency status based on the cache state .
4. **Decouple Traversal:** During ray traversal, use the octree's leaf-node size $L$ for empty-space skipping decisions, while accessing data via the brick size $B$ [S1, unofficial].
   - If a leaf node is empty, skip the entire region defined by $L$, even if it spans multiple bricks of size $B$.
5. **Handle Anisotropy (Optional):** If $L$ is anisotropic (e.g., $8 \times 8 \times 4$), adjust the traversal step sizes along axes to match the leaf dimensions, optimizing for anisotropic volume structures [S1, unofficial].

**Parameters:**
- Brick size $B$: $32^3$ voxels [S1, unofficial].
- Leaf-node size $L$: $32^3$, $16^3$, $8^3$, or $8 \times 8 \times 4$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** The leaf-node size is "approximate" [S1, unofficial]. Implementation must handle boundary mismatches between brick grids and octree leaf boundaries.
- **Anisotropy:** Using non-cubic leaf nodes (e.g., $8 \times 8 \times 4$) requires careful handling of ray intersection tests to avoid artifacts in anisotropic volumes [S1, unofficial].
- **Cache Coherence:** Since the octree and brick hierarchy are decoupled, cache updates must correctly map brick-level changes to octree leaf-level residency flags .

## Subtopics

### [Multi-level, multi-resolution page table hierarchy](20614-multi-level-multi-resolution-page-table-hierarchy.md)
Decouples spatial subdivision from resolution levels using separate page table hierarchies for each resolution level, enabling scalable address translation for extreme-scale data. Choose when rendering quality and memory scalability need independent optimization.

## Operating characteristics
- **Time Complexity:** Traversal depth is determined by leaf-node size $L$. Smaller $L$ increases traversal steps but improves empty-space skipping [S1, unofficial].
- **Memory Complexity:** Octree size depends on $L$. Finer $L$ increases node count but reduces wasted space in empty regions [S1, unofficial].
- **Performance:** Achieves higher frame rates than coupled methods by enabling efficient empty-space skipping with finer granularity than bricking [S1, unofficial].
- **Strengths:** Arbitrary spatial subdivision benefits anisotropic volumes [S1, unofficial].
- **Limitations:** Requires managing two independent hierarchies (brick and octree), increasing implementation complexity [S1, unofficial].
- **When to Use:** When empty-space skipping efficiency is critical and data has anisotropic structures or varying resolution needs [S1, unofficial].
- **When Not to Use:** When simplicity of implementation is prioritized over performance gains [S1, unofficial].

## Background
Originated in research on scalable web-based volume rendering of large biomedical datasets [S1, unofficial]. The technique builds on the observation that efficient empty-space skipping typically requires finer granularity than bricking [S1, unofficial]. It extends the residency octree update mechanism  by allowing independent configuration of traversal and data access granularities.

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
