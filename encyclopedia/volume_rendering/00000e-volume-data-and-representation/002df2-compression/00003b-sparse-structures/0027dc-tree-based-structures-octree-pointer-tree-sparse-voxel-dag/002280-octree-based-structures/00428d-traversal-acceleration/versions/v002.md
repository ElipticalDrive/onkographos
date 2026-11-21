# Traversal Acceleration


## Overview
Traversal Acceleration refers to a family of techniques that utilize hierarchical data structures, primarily octrees, to optimize the access patterns and computational cost of volume rendering and geometric queries. It solves the problem of excessive processing time caused by traversing large, sparse, or complex volumetric datasets by enabling the skipping of empty or irrelevant regions. What sets this family apart from its siblings is the specific application of acceleration structures (such as min-max octrees, summed area tables, or shadow grids) to bypass expensive per-voxel computations during rendering or intersection tests, rather than just storing the data itself.

## Shared principles
The parent article [Octree-based structures](2792-octree-based-structures.md) defines the shared mechanism for all structures in this family. Traversal Acceleration builds on [Octree-based structures](2792-octree-based-structures.md) by adding specific metadata or auxiliary structures to the octree nodes to facilitate faster culling or aggregation during the traversal phase.

## Subtopics

### [Domain-based Octree with Range Bounding Boxes](2806-domain-based-octree-with-range-bounding-boxes.md)
An acceleration structure for fiber surface extraction in tetrahedral meshes, utilizing an octree over the spatial domain to store interval ranges (RBBs) at each node. It solves the problem of efficiently identifying tetrahedra that intersect a given FSCP (Fiber Surface Control Polygon) edge by filtering candidates via 2-D intersection tests in range space. Unlike its sibling [Domain-based BVH with Range Bounding Boxes](2808-domain-based-bvh-with-range-bounding-boxes.md), the octree is strictly domain-based, which may result in more returned tetrahedra and larger memory requirements.

### Fast Classification Shear-Warp
#### definition
A fast classification shear-warp method that uses a min-max octree and a summed area table to skip transparent voxels, enabling interactive classification [S1].

#### layout
Acceleration structures: Uses a min-max octree and a summed area table, implemented as templated classes [S1]. Function signatures: Implemented as templated functions named `CompositeIntermediateNearestUnclassified` and `CompositeIntermediateLinearUnclassified` [S1].

### Octree Space Leaping
#### step
Build an octree data structure hierarchically containing important regions [S2, book]. For object-order rendering: Traverse only nonempty leaf nodes of the octree during rendering [S2, book]. For image-order ray casting: Cast rays through leaf nodes, using the regular structure of the octree to quickly step over empty nodes [S2, book].

#### layout
Octree structure: Root node contains entire volume; has eight child nodes, each representing 1/8 of the volume. Subdivision continues recursively until a node represents a homogeneous region [S2, book].

### Grid-based Shadow Ray Acceleration
#### definition
A technique to reduce the number of shadow rays cast in scenes with many light sources by using a grid to store shadow information per voxel [S3, unofficial].

#### step
Store shadow status in the grid for each light source per voxel to determine if shadow rays are necessary, thereby avoiding their execution [S3, unofficial].

#### parameter
Storage cost: two bits per light source per voxel. One bit indicates the voxel is completely in shadow; the other indicates nothing blocks the voxel from seeing the light source [S3, unofficial].

## Background
The concept of space leaping using octrees was introduced to avoid processing regions of a volume that do not contribute to the final image [S2, book]. Early implementations focused on object-order and image-order ray casting to skip empty homogeneous regions [S2, book]. The application of min-max octrees and summed area tables for fast classification shear-warp rendering was implemented in VTK to enable interactive classification by skipping transparent voxels [S1]. Grid-based acceleration for shadow rays was proposed to handle scenes with many light sources efficiently [S3, unofficial].

## Sources
[S1] Shear Warp, https://www.cg.tuwien.ac.at/courses/Visualisierung/2002-2003/Beispiel2/2-BrucknerSt-SeemannR/index2.htm, fetched 2026-10-29
[S2] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
[S3] A Fast Voxel Traversal AlgorithmforRay Tracing, https://devpress.csdn.net/v1/article/detail/102420463, fetched 2026-10-26 (unofficial)
