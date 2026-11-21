# Position-Normal Tree

Also called: Position-Normal Tree Hierarchy, Position-Normal Tree for Hierarchical Pruning

## Overview
A binary tree hierarchy over a triangle mesh used to enable hierarchical pruning of refractive path searches. It solves the performance bottleneck of testing all boundary triangles for refractive connections in large meshes by allowing entire subtrees to be discarded if they cannot contain a valid solution. It sets itself apart from flat triangle lists by maintaining bounding cones over shading normals alongside position bounding boxes, enabling geometric tests that exploit the limited bending angle of refraction.

## How it works
**Inputs and Outputs:**
- Input: A triangle mesh boundary.
- Output: A binary tree where leaves are triangles and internal nodes contain spatial and normal bounds.

**Data Structures:**
The tree is a binary hierarchy.
- **Leaf Node:** Represents a single triangle.
- **Internal Node:** Contains:
    - `aabb`: A 3D axis-aligned bounding box (AABB) covering the positions of all descendant triangles.
    - `cone`: A bounding cone covering the shading normals of all descendant triangles.
- **Memory:** Approximately 100 bytes per boundary triangle in the implementation [S1].

**Construction:**
1. Construct the hierarchy using a bottom-up approach [S1, S2].
2. Group triangles into pairs or sets.
3. Compute the AABB and bounding cone for the group.
4. Recursively repeat until a single root node remains.
5. The build time is fast; for the glass tile model, it takes 6ms [S1].
6. The tree only needs to be rebuilt if the geometry is deformed [S1].

**Traversal and Pruning:**
To find refractive connection paths, perform a depth-first traversal of the tree. At each node, test for pruning using three tests. If any test proves the subtree has no solutions, prune it [S1].

1. **Spindle Test:** A single refraction can only bend light by a limited amount. Even for arbitrarily large refractive indices $\eta$, the light can be bent by at most a right angle ($90^\circ$). For reasonable $\eta$ values, the bending is more restricted. This test checks if the required bending angle exceeds the physical limits imposed by the node's normal bounds [S1].
2. **Sidedness Test:** Checks if the geometric orientation of the node's normals relative to the incident ray allows for refraction into the medium. If all normals in the cone face away from the direction of propagation, the node is pruned [S1].
3. **Cone Overlap Test:** Checks if the bounding cone of the node's normals overlaps with the cone of possible refracted directions derived from the incident ray and the refractive index. If there is no overlap, the node is pruned [S1].

**Parameters:**
- Refractive index $\eta$: Typically 1.5 for homogeneous media in examples [S1].

**Edge Cases and Pitfalls:**
- The heuristic may miss a few paths near caustic cusps that interval-based refinement tests find, affecting at most a few pixels per image [S1].
- Without hierarchical pruning, performance degrades significantly (e.g., 32x slowdown in the pool scene) [S1].

## Subtopics

### [Split-and-Prune for Sub-Triangle Search](6272-split-and-prune-for-sub-triangle-search.md)
A recursive strategy to find refraction points on triangles with smooth normals by progressively splitting the triangle into smaller sub-triangles and pruning those that cannot contain a solution.

## Operating characteristics
- **Time Complexity:** The build time is fast (e.g., 6ms for the glass tile model) [S1]. Traversal performance scales with the effectiveness of pruning.
- **Memory Complexity:** Approximately 100 bytes per boundary triangle [S1].
- **Measured Performance:** In the pool scene, rendering took 59.4s with the hierarchy and heuristic, 141.1s with interval refinement, and 1934.6s without hierarchical pruning (32x slowdown) [S1].
- **Strengths:** Essential for performance on meshes with many triangles. Enables efficient single-scattering calculations in refractive media.
- **Limitations:** Requires rebuilding if geometry deforms. The heuristic may miss rare paths near caustic cusps compared to interval-based methods [S1].
- **When to use:** When rendering single scattering in refractive media with triangle mesh boundaries, especially for meshes with more than a few triangles.

## Background
The Position-Normal Tree was introduced by Bala et al. [S1, S2], building on earlier work by Snyder [S2] and Walter et al. [S2]. It is used in the context of single scattering in refractive media with triangle mesh boundaries to efficiently compute refractive connection paths through a mesh [S1].

## Sources
[S1] Single Scattering in Refractive Media with Triangle Mesh Boundaries, https://doi.org/10.1145/1531326.1531398, fetched 2026-10-26
