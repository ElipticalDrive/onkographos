# Volume BVH
Also called: Volume Bounding Volume Hierarchy, Scene-level Volume BVH
## Overview
Volume BVH is a hierarchical acceleration structure for volume rendering that handles multiple objects, instances, and transformations by using bounding boxes to quickly identify which volume instances overlap a given ray segment [S1, unofficial]. It solves the problem of efficiently determining which potentially transformed or instanced volume sources contribute to a specific ray, serving as a coarse-level filter before detailed voxel traversal [S1, unofficial]. It sets itself apart from sibling methods by focusing on the scene-level management of multiple distinct volume objects and instances, rather than optimizing the construction of a single object's hierarchy or handling specific primitive types like Gaussian surfels.
## Shared principles
The family relies on a tree of axis-aligned bounding boxes (AABBs) where each node contains a bounding volume enclosing its children [S1, unofficial]. The core mechanism involves traversing this tree to find all leaf nodes (representing volume instances or objects) that intersect a ray segment defined by points A and B [S1, unofficial].

Key shared properties:
1.  **Bounding Volumes**: Nodes store AABBs. These can represent full object bounds or finer inner/leaf node bounds to reduce false positives [S1, unofficial].
2.  **Instance Handling**: The structure must robustly detect overlap for objects that are rotated, scaled, or instanced multiple times [S1, unofficial].
3.  **Gathering**: The traversal result is a list of overlapping object instances, which is gathered into an array in the kernel before volume shading begins [S1, unofficial].
4.  **Orthogonality**: The BVH operates at a level orthogonal to the internal voxel traversal (e.g., HDDA), which may be used subsequently for fine-grained stepping [S1, unofficial].

## Subtopics

### [BVH for Gaussian Surfels](17858-bvh-for-gaussian-surfels.md)
Optimizes BVH construction and traversal specifically for Gaussian surfel primitives, differing from the general object-instance focus of the main Volume BVH.

### [Heuristic BVH Construction (Top-Down with Plane-Sweep)](18008-heuristic-bvh-construction-top-down-with-plane-sweep.md)
Provides a specific algorithm for building the BVH tree using plane-sweep heuristics, which can be applied to Volume BVH construction but is not unique to it.

## Background
The concept was proposed for Blender's Cycles renderer to optimize
