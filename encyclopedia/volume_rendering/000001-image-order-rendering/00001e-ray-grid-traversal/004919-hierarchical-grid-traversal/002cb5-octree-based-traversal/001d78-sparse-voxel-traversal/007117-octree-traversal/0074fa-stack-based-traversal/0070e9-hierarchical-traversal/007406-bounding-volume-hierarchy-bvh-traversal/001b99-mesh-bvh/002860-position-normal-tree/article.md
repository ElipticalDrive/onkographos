# Position-Normal Tree

Also called: Position-Normal Tree Hierarchy, Position-Normal Tree for Hierarchical Pruning

## Overview
A binary tree hierarchy over a triangle mesh where leaves are triangles and internal nodes contain a 3D axis-aligned bounding box (AABB) over positions and a bounding cone over shading normals for all descendants [S1]. It stores position and normal information for boundary triangles to enable hierarchical pruning of refractive path searches [S1]. It is distinguished from interval-based refinement by prioritizing speed over guaranteed convergence, typically running 2 to 5 times faster while producing visually equivalent results [S1].

## How it works
**Inputs:** A triangle mesh boundary [S1].
**Outputs:** A binary tree structure for pruning refractive path searches [S1].
**Data Structures:** Each node contains an AABB over positions, a bounding cone over normals, child pointers for internal nodes, and a triangle reference for leaves [S1]. Memory usage is approximately 100 bytes per boundary triangle [S1].
**Steps:** 1. Construct the hierarchy using a bottom-up approach [S1]. 2. Perform a depth-first traversal of the tree [S1]. 3. At each node, test for pruning using the spindle test, sidedness test, and cone overlap test [S1]. 4. If any test proves no solution exists, prune the subtree [S1]. The exact merging algorithm is not given in the sources.
**Formulas:** Not given in the sources.
**Parameters:** Refractive index ($\eta$) is typically 1.5 for homogeneous media in the examples [S1].
**Edge Cases and Pitfalls:** The tree only needs to be rebuilt if the geometry is deformed [S1]. The pruning tests are heuristic and may miss a few paths near caustic cusps on rare occasions, affecting at most a few pixels per image compared to interval-based refinement [S1].

## Subtopics

### [Split-and-Prune for Sub-Triangle Search](6272-split-and-prune-for-sub-triangle-search.md)
A recursive strategy to find refraction points on triangles with smooth normals by progressively splitting the triangle into smaller sub-triangles and pruning those that cannot contain a solution. It is distinguished from interval-based refinement by prioritizing speed over guaranteed convergence, typically running 2 to 5 times faster while producing visually equivalent results. Choose it when performance is critical and visually equivalent results are acceptable over strict convergence guarantees.

## Operating characteristics
Essential for performance on meshes with many triangles [S1]. Without hierarchical pruning, the pool scene took 1934.6s versus 59.4s with it, a 32x slowdown [S1]. Build time is very fast, taking 6ms for the glass tile model [S1]. Memory usage is approximately 100 bytes per boundary triangle [S1]. The position-normal tree and hierarchical pruning are essential for getting good performance on boundary meshes with more than a few triangles [S1].

## Background
The position-normal tree was introduced by Bala et al. [S1], building on earlier work by Snyder [S1] and Walter et al. [S1]. It is used in single scattering in refractive media with triangle mesh boundaries to efficiently compute refractive connection paths [S1].

## Sources
[S1] Single Scattering in Refractive Media with Triangle Mesh Boundaries, https://doi.org/10.1145/1531326.1531398, fetched 2026-10-26
