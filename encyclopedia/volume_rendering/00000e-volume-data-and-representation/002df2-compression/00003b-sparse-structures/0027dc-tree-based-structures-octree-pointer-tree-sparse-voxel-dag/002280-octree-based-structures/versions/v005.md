# Octree-based structures

Also called: Sparse Voxel Octrees (SVO), VDB, Implicit Octrees

## Overview
Octree-based structures represent volumetric data using a recursive spatial subdivision where empty or uniform regions are collapsed into single nodes, providing compression proportional to data sparsity [G]. This family solves the memory and bandwidth bottlenecks of dense voxel grids by enabling efficient culling of empty space and adaptive resolution [G]. Unlike sibling structures like Pointer Trees or Sparse Voxel DAGs, standard octrees rely on a fixed 8-way branching factor and explicit or implicit parent-child pointers, prioritizing hierarchical traversal and spatial locality over the deduplication or pointer-free compactness of their variants [G].

## Shared principles
All structures in this family share the recursive subdivision, sparsity encoding, hierarchical traversal, and spatial locality mechanisms defined in [Tree-based structures (Octree, Pointer Tree, Sparse Voxel DAG)](4139-tree-based-structures-octree-pointer-tree-sparse-voxel-dag.md).

## Subtopics

### [Implicit Octree Storage](939-implicit-octree-storage.md)
Implicit Octree Storage is a serialization method for sparse voxel octrees that eliminates explicit node pointers by sorting tree nodes into a linear array using a deterministic spatial order, such as breadth-first or Morton order. It sets itself apart from pointer-based variants by prioritizing compact storage over random-access speed, as the implicit ordering is explicitly unsuitable for efficient traversal. Choose this method when persistent storage, streaming, or bandwidth-constrained environments are the primary constraints.

### [Implicit Octree Traversal](1220-implicit-octree-traversal.md)
Implicit Octree Traversal is a method for building and traversing a sparse octree representation of a distance field to extract isosurfaces, specifically designed for dual contouring and marching cubes algorithms. It sets itself apart from sibling methods by focusing on bitwise arithmetic mapping for neighbor retrieval rather than seed placement, and differs from geometric neighbor-finding techniques by relying on direct coordinate hashing. Choose this method when reconstructing 2x2x2 cells for isosurface primitives from sparse distance fields without storing explicit parent pointers.

### [OpenVDB (VDB)](1508-openvdb-vdb.md)
OpenVDB is a sparse voxel representation used for the hierarchical encoding and compression of volumetric grids, featuring fast $O(1)$ random access for both modification and retrieval operations. It sets itself apart from implicit octree variants by using explicit 64-bit direct access bit masks at every level of a shallow, four-level tree structure, enabling fast runtime modifications and out-of-core streaming. Choose this method for managing large, sparse volumetric datasets like fluid simulations where frequent in-memory updates are required.

### [Time-Variant Volume Rendering with Octrees](2232-time-variant-volume-rendering-with-octrees.md)
This method enables the interactive rendering of time-variant volume datasets by storing multiple timesteps in limited CPU main memory using octree compression. It sets itself apart from standard sparse structures by explicitly exploiting temporal coherence to compress full sequences of medium-sized time-variant data, allowing the renderer to skip transparent or uniform blocks across timesteps. Choose this method when GPU memory capacity and bus bandwidth limit the number of timesteps that can be stored in-core.

### [Domain-based Octree with Range Bounding Boxes](2806-domain-based-octree-with-range-bounding-boxes.md)
The Domain-based Octree with Range Bounding Boxes is an acceleration structure for fiber surface extraction in tetrahedral meshes, utilizing an octree over the spatial domain to store interval ranges (RBBs) at each node. It sets itself apart from its sibling BVH variant by using a strictly domain-based structure, which may return more tetrahedra but offers a simpler hierarchical structure based on regular spatial subdivision. Choose this method when efficiently identifying tetrahedra intersecting a given FSCP edge via conservative 2-D range space intersection tests is required.

### [Tree64](3486-tree64.md)
Tree64 is a sparse voxel tree data structure with a 4³ branching factor, designed to reduce tree depth and per-voxel overhead compared to standard octrees. It sets itself apart from standard octrees by utilizing compact 12-byte node layouts and optimized GPU traversal kernels that leverage relative pointers and depth-first Morton ordering. Choose this method for real-time ray tracing of large, sparse voxel scenes where traversal throughput and memory bandwidth utilization are critical.

#### Dynamic Adaptive Octree Grid (DAOG)
**Definition:** A voxel representation structure that offers significant memory savings compared to Regular Grid and standard Octree, allowing representation of large surfaces with little space [S1, unofficial].
**Memory Usage:** For a 1024×1024×1024 Cornell Box model: Regular Grid uses 1048576 KB, Octree uses 158087 KB, DAOG uses 269 KB. For a 1024×1024×1024 Batman model: Regular Grid uses 1048576 KB, Octree uses 106303 KB, DAOG uses 813 KB [S1, unofficial].
**Construction Time:** For 1024×1024×1024 Cornell Box: Octree takes 5.153s, DAOG takes 67.680s, DAOG serialized takes 0.142ms [S1, unofficial].

#### Tensor Approximation Rendering
**Definition:** Rendering method using tensor approximation combined with a multiresolution octree hierarchy, decompressing the full working set before rendering [S2].
**Steps:** Reconstruction is based on the Tucker formulation. Implement tensor times matrix (TTM) multiplications. GPU Kernel: Each kernel thread decompresses one voxel [S2].
**Formula:** Computational complexity per voxel: $O(M)$, where $M$ is the rank. This grows linearly with $M$, instead of $O(M^3)$ [S2].
**Comparison:** Differs from naive tensor reconstruction by reducing complexity from cubic to linear in rank $M$ via optimized kernel threading [S2].
**Usage:** Proposed by Suter et al. [SIM*11] [S2].

#### Vector-based Octree Storage
**Definition:** Storing octree nodes as `vector3` structures where $x, y, z$ represent the center point and $w$ is a flags field [S3, unofficial].
**Steps:** Use the $w$ component flags field to denote which of the 8 octant child nodes follow the current node, requiring 8 bits. Store entities as bounding boxes using $r, g, b$ for dimensions and $w$ for other data. Define a special vector (e.g., if $w + z$ equals a magic value) to indicate that an object list follows. Walk the octree and output vectors to a memory buffer [S3, unofficial].
**Parameter:** A 512×512 texture can hold a fully packed octree 5 levels deep (32,768 nodes) with 8 objects each, or a 4-level octree with 64 objects each [S3, unofficial].

#### Fast Classification Shear-Warp
**Definition:** A fast classification shear-warp method that uses a min-max octree and a summed area table to skip transparent voxels, enabling interactive classification [S4].
**Layout:** Acceleration structures: Uses a min-max octree and a summed area table, implemented as templated classes. Function signatures: Implemented as templated functions named `CompositeIntermediateNearestUnclassified` and `CompositeIntermediateLinearUnclassified` [S4].

## Background
Not given in the sources.

## Sources
[S1] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
[S2] PDF State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S3] Efficient storage for a sparse octree?, https://stackoverflow.com/questions/6085209/efficient-storage-for-a-sparse-octree, fetched 2026-10-26 (unofficial)
[S4] Shear Warp, https://www.cg.tuwien.ac.at/courses/Visualisierung/2002-2003/Beispiel2/2-BrucknerSt-SeemannR/index2.htm, fetched 2026-10-29
