# Traversal Acceleration


## Overview
Traversal Acceleration refers to a family of techniques that use hierarchical data structures, primarily octrees, to skip empty or homogeneous regions of a volume during rendering or data extraction, thereby reducing computational cost. It solves the problem of inefficiently processing large, sparse, or complex volumetric datasets by avoiding redundant calculations in areas that do not contribute to the final image or result. Unlike sibling methods that may rely on implicit storage or specific terrain representations, this family focuses on the traversal logic and acceleration structures (like min-max octrees or range bounding boxes) to filter candidates or skip steps.

## Shared principles
All structures in this family share the recursive subdivision, sparsity encoding, hierarchical traversal, and spatial locality mechanisms defined in [Tree-based structures (Octree, Pointer Tree, Sparse Voxel DAG)](4139-tree-based-structures-octree-pointer-tree-sparse-voxel-dag.md).

## Subtopics

### [Domain-based Octree with Range Bounding Boxes](2806-domain-based-octree-with-range-bounding-boxes.md)
The Domain-based Octree with Range Bounding Boxes is an acceleration structure for fiber surface extraction in tetrahedral meshes, utilizing an octree over the spatial domain to store interval ranges (RBBs) at each node. It solves the problem of efficiently identifying tetrahedra that intersect a given FSCP (Fiber Surface Control Polygon) edge by filtering candidates via 2-D intersection tests in range space. Unlike its sibling [Domain-based BVH with Range Bounding Boxes](2808-domain-based-bvh-with-range-bounding-boxes.md), the octree is strictly domain-based, which may result in more returned tetrahedra and larger memory requirements.

### Fast Classification Shear-Warp
#### definition
- [S1] A fast classification shear-warp method that uses a min-max octree and a summed area table to skip transparent voxels, enabling interactive classification.
#### layout
- [S1] Acceleration structures: Uses a min-max octree and a summed area table, implemented as templated classes.
- [S1] Function signatures: Implemented as templated functions named `CompositeIntermediateNearestUnclassified` and `CompositeIntermediateLinearUnclassified`.

### Octree Space Leaping
#### step
- [S2, book] Build an octree data structure hierarchically containing important regions.
- [S2, book] For object-order rendering: Traverse only nonempty leaf nodes of the octree during rendering.
- [S2, book] For image-order ray casting: Cast rays through leaf nodes, using the regular structure of the octree to quickly step over empty nodes.
#### layout
- [S2, book] Octree structure: Root node contains entire volume; has eight child nodes, each representing 1/8 of the volume. Subdivision continues recursively until a node represents a homogeneous region.

### Grid-based Shadow Ray Acceleration
#### definition
- [S3, unofficial] A technique to reduce the number of shadow rays cast in scenes with many light sources by using a grid to store shadow information per voxel.
#### step
- [S3, unofficial] Store shadow status in the grid for each light source per voxel to determine if shadow rays are necessary, thereby avoiding their execution.
#### parameter
- [S3, unofficial] Storage cost: two bits per light source per voxel. One bit indicates the voxel is completely in shadow; the other indicates nothing blocks the voxel from seeing the light source.

## Background
The concept of space leaping using octrees was developed to take advantage of the observation that many regions of a volume do not contribute to the final image [S2, book]. The technique involves building an octree data structure that hierarchically contains all important regions, allowing for efficient skipping of empty regions during rendering [S2, book]. Fast classification shear-warp algorithms were introduced to enable interactive classification by using min-max octrees and summed area tables to skip transparent voxels [S1]. Grid-based shadow ray acceleration was proposed as a method to reduce the computational cost of shadow rays in scenes with many light sources by storing shadow status in a grid [S3, unofficial].

## Sources
[S1] Shear Warp, https://www.cg.tuwien.ac.at/courses/Visualisierung/2002-2003/Beispiel2/2-BrucknerSt-SeemannR/index2.htm, fetched 2026-10-29
[S2] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
[S3] A Fast Voxel Traversal AlgorithmforRay Tracing, https://devpress.csdn.net/v1/article/detail/102420463, fetched 2026-10-26 (unofficial)
