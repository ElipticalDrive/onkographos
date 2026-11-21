# Bounding Volume Hierarchy (BVH) Traversal

Also called: BVH Traversal

## Overview
Bounding Volume Hierarchy (BVH) traversal accelerates ray-scene intersection and distance queries by recursively testing rays against a binary tree of Axis-Aligned Bounding Boxes (AABBs) [S1, unofficial]. It is distinguished from hierarchical grid methods (like Octree Traversal) by operating directly on unstructured geometry (e.g., triangle meshes) rather than voxelized volumes, and from software-only BVHs by leveraging hardware acceleration (e.g., NVIDIA OptiX) for ray-triangle intersections .

## Shared principles
The BVH is a binary tree where internal nodes contain AABBs enclosing their descendants, and leaves contain primitive geometry (typically triangles) [S1, unofficial]. Traversal begins at the root. For each node, the distance from the query point (or ray origin) to the node's AABB is computed. If this distance exceeds the current best distance $d_{min}$ (for point queries) or the ray does not intersect the AABB (for ray queries), the node is pruned [S1, unofficial]. If a leaf is reached, the exact primitive intersection or distance is computed, and $d_{min}$ is updated if a closer primitive is found [S1, unofficial].

## Subtopics

### [Mesh BVH](1062-mesh-bvh.md)
Mesh BVH is a GPU-accelerated BVH designed for triangle meshes to support efficient distance queries, signed distance field (SDF) computation, and ray intersection tests. It is distinguished from hierarchical grid methods by operating on unstructured triangle geometry and from software-only BVHs by leveraging NVIDIA OptiX hardware acceleration. Choose this for SDF computation or collision detection on complex meshes.

### [Initial Upper Bound Estimation](15442-initial-upper-bound-estimation.md)
Initial Upper Bound Estimation computes a tight initial upper bound to prune BVH-tree traversal during distance field computation for isosurface approximation. It differs from standard hierarchical traversal by proactively sampling the approximated surface to establish a pruning threshold before traversal begins, rather than waiting for the first leaf encounter. Choose this when traversing BVHs for isosurface distance fields to reduce node visits.

### Coarse Mesh Traversal via BVH
**Definition**
Uses a BVH built over a coarse tetrahedral mesh to efficiently locate the coarse element containing a sample point [S1, unofficial].

**Steps**
1. Build a triangle BVH over the coarse mesh by inserting four triangular faces facing outside for every coarse element [S1, unofficial].
2. Trace rays into the BVH using OptiX Wrapper Library (OWL) hit and miss functions [S1, unofficial].
3. The hit function returns the hit point and the coarse tetrahedron ID [S1, unofficial].
4. The miss function reports no intersection with BVH faces [S1, unofficial].
5. Trace two rays: one with front-face culling and one with back-face culling [S1, unofficial]. If the front-culling ray misses, the point is outside the coarse mesh. If the back-culling ray misses, the point is inside the last intersected element [S1, unofficial].

**Comparison**
Unlike element marching which uses connectivity, point location querying skips coarse elements not being sampled and allows varying refinement levels in the coarse mesh [S1, unofficial].

### [Multi-Resolution Rendering with K-D Tree](1942-multi-resolution-rendering-with-k-d-tree.md)

### [Leaf Collection](899-leaf-collection.md)

### [Point Plane Test Traversal](982-point-plane-test-traversal.md)

### [Coarse Mesh Traversal](3062-coarse-mesh-traversal.md)

### [Node Omission in Ray-Tree Traversal](3305-node-omission-in-ray-tree-traversal.md)

### [Two-Pass Hierarchy Traversal](3339-two-pass-hierarchy-traversal.md)

### [Adaptive Out-of-Core Single-Pass GPU Raycasting](3440-adaptive-out-of-core-single-pass-gpu-raycasting.md)

## Background
BVH traversal for point location and distance queries in volume rendering contexts has been applied to tetrahedral adaptive mesh refinement (TET-AMR) data [S1, unofficial]. The approach uses a BVH to locate coarse elements efficiently, allowing the renderer to skip elements not being sampled and support varying refinement levels in the coarse mesh [S1, unofficial]. Implementation often utilizes NVIDIA OptiX for hardware-accelerated ray-triangle intersection tests [S1, unofficial].

## Sources
[S1] Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data, https://repository.bilkent.edu.tr/items/fcbc592a-2646-49af-8b84-67308b6fd3e9, fetched 2026-10-27 (unofficial)
