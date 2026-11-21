# Tetrahedral Primitive Representation


## Overview
Tetrahedral Primitive Representation is a family of volume rendering techniques that use tetrahedra as the fundamental geometric primitives for projection and compositing. It addresses the rendering of unstructured grids, point clouds, and differentiable scenes by decomposing the volume into tetrahedral cells [S1, S2, S3]. It sets itself apart from sibling methods by focusing on primitive-level data structures (such as strips, fans, or specific basis vectors) and the explicit projection of these tetrahedra to screen space [S1, S2][S4, unofficial].

## Shared principles
This family builds on the shared principles of [Tetrahedral representation and traversal](4396-tetrahedral-representation-and-traversal.md).

## Subtopics

### [Tetrahedral Strips and Fans](1389-tetrahedral-strips-and-fans.md)
Specialized graphics primitives designed to reduce the bandwidth bottleneck between main memory and the graphics processor when rendering unstructured tetrahedral meshes. They set themselves apart by grouping tetrahedra into sequences (strips or fans) that share geometric elements, allowing hardware vertex reuse. Choose this method when rendering unstructured tetrahedral meshes on GPUs to minimize data transmission overhead.

### [Marching Tetrahedrons Algorithm](940-marching-tetrahedrons-algorithm.md)
An isosurface extraction algorithm that operates on tetrahedral cells. It sets itself apart by generating polygonal surfaces (triangles) from scalar field data rather than performing direct volume compositing. Choose this method when surface extraction is required instead of direct volume rendering.

### [Adaptive Delaunay Update](4923-adaptive-delaunay-update.md)
Focuses on the dynamic generation and refinement of the tetrahedral mesh. It sets itself apart by handling the topological changes required to maintain a Delaunay triangulation as the dataset or resolution changes. Choose this method when working with evolving or adaptive datasets that require continuous mesh refinement.

### [Grid-based Traversal](7083-grid-based-traversal.md)
Optimizes the search for tetrahedra intersected by rays using spatial acceleration structures. It sets itself apart by focusing on the efficiency of finding relevant primitives in large datasets. Choose this method when rendering large-scale volumetric data where traversal performance is critical.

### [Parallel Projected Tetrahedra with KD-Tree Partitioning](8767-parallel-projected-tetrahedra-with-kd-tree-partitioning.md)
Extends the projection method to multi-core or GPU architectures using KD-tree partitioning. It sets itself apart by distributing the projection and compositing workload across parallel processing units. Choose this method when high-performance rendering on parallel hardware is required.

### [Tetrahedral Mesh Connectivity Encoding](8957-tetrahedral-mesh-connectivity-encoding.md)
Deals with the compact storage and retrieval of adjacency information between tetrahedra. It sets itself apart by optimizing the data layout for efficient neighbor lookups during traversal or strip generation. Choose this method when memory bandwidth or cache efficiency for mesh connectivity is a bottleneck.

### [Delaunay Triangulation for Tetrahedral Representation](9905-delaunay-triangulation-for-tetrahedral-representation.md)
Provides the mathematical foundation for generating the tetrahedral mesh from a set of points. It sets itself apart by ensuring optimal geometric properties (such as maximizing the minimum angle) for the resulting tetrahedra. Choose this method when generating high-quality tetrahedral meshes from point clouds.

### [TetraSDF](11632-tetrasdf.md)
Represents signed distance fields using tetrahedral primitives. It sets itself apart by encoding geometric distance information within the tetrahedra, enabling high-quality surface reconstruction and differentiable rendering. Choose this method when differentiable rendering or precise surface reconstruction is needed.

### Point Representation via Tetrahedra
Representing data points as small tetrahedra to fit them into the triangular mesh framework for rendering [S1]. The `size` parameter specifies the size of the point tetrahedra relative to the data ranges [S1].

### LinPrim Tetrahedron Representation
A scene representation using transparent tetrahedra bounded by triangular faces with homogeneous density [S2]. It defines four basis vectors describing directions from the center on which corners are located, equally spaced to represent regular tetrahedra, and optimizes the distance between the center and each corner independently [S2]. Each primitive requires 12 floats per primitive (omitting color) due to the absence of symmetry in the formulation [S2]. Unlike octahedra, the optimized position (center) can differ from the geometric center of the primitive due to lack of symmetry [S2].

### Tetrahedral Slicing
A rendering algorithm used by TMRenderAction that represents volumetric geometry using tetrahedra as the basic unit [S3]. It tessellates the given volumetric geometry into a tetrahedral mesh, sorts the mesh in a back-to-front visibility order, sets the OpenGL state for a given shader, and starts with the rearmost element to slice the tetrahedra one-by-one and render the generated polygonal geometry [S3].

### Triangle Fan Optimization
Using triangle fans instead of individual triangles or strips to reduce the number of vertices transmitted and OpenGL procedure calls, leveraging the fact that split cases in projected tetrahedra can directly create triangle fans [S4, unofficial]. Each new vertex specified with `glVertex` specifies a new triangle. On the NASA Langley fighter dataset with 70,125 tetrahedra, there is an average of 3.4 triangles per tetrahedron (Class 1: 60% 3 triangles, Class 2: 40% 4 triangles) [S4, unofficial]. For $n$ tetrahedra, there are $10 \times 8n$ procedure calls with fans versus $20 \times 4n$ procedure calls without fans, a factor of 2 reduction [S4, unofficial].

## Background
The method is applied to render point clouds by representing data points as small tetrahedra [S1], to enable differentiable volumetric rendering with linear primitives [S2], and for real-time volume visualization via TMRenderAction [S3]. Triangle fan optimizations for projected tetrahedra reduce OpenGL procedure calls by a factor of 2 [S4, unofficial].

## Sources
[S1] Computing and Displaying Isosurfaces inR, https://doi.org/10.18637/jss.v028.i01, fetched 2026-10-27
[S2] LinPrim: Linear Primitives for Differentiable Volumetric Rendering, https://arxiv.org/abs/2501.16312, fetched 2026-10-26
[S3] Chapter 4. Texture Mapping Render Action, https://techpubs.jurassic.nl/library/manuals/4000/007-4389-010/sgi_html/ch04.html, fetched 2026-10-27
[S4] Volume Visualization and Volume Rendering Techniques, https://www.cs.unh.edu/~cs880/visualization/Meissner-VolRenderingEGTutorial-rdb.pdf, fetched 2026-10-27 (unofficial)
