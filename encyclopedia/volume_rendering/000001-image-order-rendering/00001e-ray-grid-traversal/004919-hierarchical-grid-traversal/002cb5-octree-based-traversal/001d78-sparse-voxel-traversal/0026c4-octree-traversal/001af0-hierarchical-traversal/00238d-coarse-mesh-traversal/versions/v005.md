# Coarse Mesh Traversal


## Overview
Coarse Mesh Traversal is a hierarchical rendering technique that accelerates volume rendering by decoupling the global search for active data regions from the local sampling of fine-grained geometry. It solves the problem of inefficiently traversing large, sparse, or multi-resolution datasets by first identifying which coarse elements (macrocells) a ray intersects, and then performing detailed rendering only within those specific regions. What sets it apart from its siblings is the explicit construction of a coarse mesh (typically a BVH) from the underlying adaptive or hierarchical data structure, allowing the renderer to skip empty or irrelevant volumes entirely before entering the fine-level traversal loop.

## Shared principles
The family shares a two-stage traversal mechanism. First, a coarse mesh is constructed from the input data, where each element represents a cluster of fine-level primitives (e.g., tetrahedra or voxels). This mesh is indexed by a spatial acceleration structure, typically a Bounding Volume Hierarchy (BVH). Second, for each ray, the algorithm performs a coarse traversal to determine the entry and exit distances ($t_{entry}, t_{exit}$) for the current coarse element. If the element is active (non-empty), the ray is then marched through the fine-level data contained within those bounds.

Key mathematical constraints include:
1.  **Boundary Precision:** To prevent re-intersection artifacts when moving from one coarse element to the next, the exit distance $t_{exit}$ is often perturbed to the next representable floating-point number using `nextafterf(t, infinity)` [S1, unofficial].
2.  **Culling:** Coarse elements are often associated with scalar statistics (e.g., maximum density) to allow early-out culling before fine-level sampling begins.
3.  **Nested Structure:** The fine-level traversal is bounded by the coarse element's intersection distances, ensuring that the inner loop does not exceed the macrocell's spatial extent.

## Subtopics

### [Coarse Mesh Traversal with Dual-Ray Culling](987-coarse-mesh-traversal-with-dual-ray-culling.md)
Uses two distinct ray traces (front-face and back-face culling) to robustly determine entry and exit boundaries, handling non-convex shapes and holes.

### [Two-level Nested Ray Marching](1045-two-level-nested-ray-marching.md)
Applies coarse traversal to Adaptive Mesh Refinement (AMR) data, using a constant step size for the inner fine-level march to maintain native resolution without uniform resampling.

### [Varying Refinement Levels of Coarse Elements](1055-varying-refinement-levels-of-coarse-elements.md)
Optimizes the coarse mesh by harvesting nodes from specific levels of the refinement tree, balancing BVH traversal cost against fine-level sampling overhead.

## Background
Coarse Mesh Traversal builds on the principles of hierarchical volume rendering, specifically leveraging the spatial coherence of adaptive meshes. It is primarily used in the visualization of scientific simulation data, such as fluid dynamics or astrophysics, where data is naturally structured in adaptive tetrahedral or hexahedral meshes. The method addresses the memory and performance bottlenecks of uniform grid resampling by operating directly on the native hierarchical structure.

## Sources
