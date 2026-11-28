# Ray-Node Intersection Logic

Also called: Ray-Octree Intersection, Ray-Node Traversal, Sparse Ray-Octree Intersection

## Overview
Ray-Node Intersection Logic defines the geometric and topological rules for determining whether a ray intersects an octree node and identifying the specific child octant entered first. It solves the problem of efficiently navigating hierarchical voxel data by replacing recursive pointer chasing with iterative parametric $t$-value calculations and stack-based backtracking. It differs from sibling methods by focusing on the low-level arithmetic of bounding box intersections and child selection logic rather than high-level rendering pipelines or neighbor-finding mechanisms.

## Shared principles
All child algorithms rely on the parametric ray representation $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ to compute intersection distances with axis-aligned bounding boxes (AABBs). The core mechanism involves:
1.  **Bounding Box Intersection:** Calculating $t_{min}$ and $t_{max}$ for the ray against the node's bounds. If $t_{min} > t_{max}$, the ray misses the node .
2.  **Child Selection:** Determining the entry octant index $k \in \{0, \dots, 7\}$ based on the ray's position relative to the node's center. This is often computed via bitwise operations on the voxel coordinates or by comparing parametric $t$-values at the node center .
3.  **Traversal State:** Maintaining the current depth, scale, and $t$-span $[t_{min}, t_{max}]$ to prune branches where the ray does not intersect the child's volume .
4.  **Backtracking:** Using an explicit stack or depth decrement to return to parent nodes when a child is empty or the ray exits the current subtree .

## Subtopics

### [Sparse Ray-Octree Intersection (DECIDE)](3006-sparse-ray-octree-intersection-decide.md)
A GPU-optimized iterative algorithm using bitwise child-slot permutation and t-span calculations. It handles traversal, descent, and backtracking via an explicit stack, suitable for parallel execution.

### [Ray-Occlusion Traversal in Octree](5121-ray-occlusion-traversal-in-octree.md)
A fixed-depth descent loop with explicit parent-boundary checks. It explicitly handles rays starting inside or outside the root bounding box via plane intersection logic, primarily used for shadow ray computation.

### [Octree Ray Traversal](11317-octree-ray-traversal.md)
An algorithm for pointer-based octrees that returns intersected octants sorted by distance. It uses parametric $t$-values to traverse the hierarchy, focusing on discrete octant traversal logic rather than dense sampling.

## Background
The concept of ray-triangle intersection is foundational, but octree traversal extends this to hierarchical bounding volumes. Early work by Williams et al. (2009) introduced efficient GPU-based sparse voxel octree traversal . Subsequent developments focused on optimizing the child selection logic and stack management for real-time rendering .

