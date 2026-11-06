# Octree-based structures

Also called: Sparse Voxel Octrees (SVO), VDB, Implicit Octrees

## Overview
Octree-based structures represent volumetric data using a recursive spatial subdivision where empty or uniform regions are collapsed into single nodes, providing compression proportional to data sparsity. This family solves the memory and bandwidth bottlenecks of dense voxel grids by enabling efficient culling of empty space and adaptive resolution. Unlike sibling structures like Pointer Trees or Sparse Voxel DAGs, standard octrees rely on a fixed 8-way branching factor and explicit or implicit parent-child pointers, prioritizing hierarchical traversal and spatial locality over the deduplication or pointer-free compactness of their variants.

## Shared principles
All structures in this family share the following mechanism:

1.  **Recursive Subdivision:** The 3D domain is recursively divided into sub-volumes. In standard octrees, each node has 8 children. In pointer trees, the branching factor $N$ varies ($N^3$ children).
2.  **Sparsity Encoding:** Empty regions are represented by null pointers or absence of children, rather than storing zero-valued voxels. This provides compression proportional to the sparsity of the data.
3.  **Hierarchical Traversal:** Rendering or access requires traversing from the root to the leaf. This allows skipping large empty volumes (culling) and adapting resolution to view distance or detail requirements.
4.  **Spatial Locality:** Children of a node are spatially contiguous. This locality is often exploited via Morton ordering (Z-order curve) to map 3D coordinates to 1D keys, enabling efficient neighbor lookups and linear array representations in implicit variants [G].

## Subtopics

### [Implicit Octree Storage](939-implicit-octree-storage.md)
Implicit Octree Storage is a memory-efficient serialization format for sparse voxel octrees that eliminates explicit node pointers by mapping tree nodes to a linear array via a deterministic spatial ordering, such as breadth-first or Morton order. It solves the problem of high memory overhead and fragmentation associated with pointer-based tree structures, making it suitable for streaming, persistent storage, and bandwidth-constrained environments. Unlike pointer-based variants or traversal-optimized structures like [Implicit Octree Traversal](1220-implicit-octree-traversal.md), this method prioritizes compact storage over random-access speed, as the implicit order is not designed for efficient on-the-fly traversal. See [Implicit Octree Storage](939-implicit-octree-storage.md).

### [Implicit Octree Traversal](1220-implicit-octree-traversal.md)
Implicit Octree Traversal is a method for constructing sparse octrees from distance fields to facilitate isosurface extraction, specifically targeting dual contouring or marching cubes on sparse data. It differs from explicit octree structures by using Morton coordinates to implicitly define the tree hierarchy, allowing for efficient point location and neighbor retrieval without storing explicit parent pointers or full node metadata. Unlike [Implicit Octree Storage](939-implicit-octree-storage.md), which focuses on the storage layout, this method defines the traversal algorithm to build the primal vertex map and locate dual vertices for mesh generation. See [Implicit Octree Traversal](1220-implicit-octree-traversal.md).

### [OpenVDB (VDB)](1508-openvdb-vdb.md)
OpenVDB is a sparse voxel representation designed for the hierarchical encoding and compression of volumetric grids, enabling fast $O(1)$ random access for both modification and retrieval. It solves the problem of managing large, sparse volumetric datasets (such as those from fluid simulations) by providing a shallow, four-level tree structure that supports efficient in-memory manipulation and out-of-core streaming. Unlike deep recursive octrees, OpenVDB uses fixed-size leaf nodes and direct-access bitmasks at every level to minimize traversal overhead, making it a standard in production rendering and visual effects software. See [OpenVDB (VDB)](1508-openvdb-vdb.md).

### [Time-Variant Volume Rendering with Octrees](2232-time-variant-volume-rendering-with-octrees.md)
This method enables the interactive rendering of time-variant volume datasets by storing multiple timesteps in limited CPU main memory using octree compression, thereby overcoming GPU memory capacity and bus bandwidth limitations. It sets itself apart from standard sparse structures by explicitly exploiting temporal coherence to compress full sequences of medium-sized time-variant data, allowing the renderer to skip transparent or uniform blocks and reducing I/O costs and inter-frame delay. Unlike baseline renderers that process full uncompressed volumes, this approach fits large multi-timestep datasets (e.g., 4.11 GB) into main memory (e.g., 678 MB) for interactive frame rates. See [Time-Variant Volume Rendering with Octrees](2232-time-variant-volume-rendering-with-octrees.md).

### [Domain-based Octree with Range Bounding Boxes](2806-domain-based-octree-with-range-bounding-boxes.md)
The Domain-based Octree with Range Bounding Boxes is an acceleration structure for fiber surface extraction in tetrahedral meshes. It solves the problem of efficiently identifying tetrahedra that intersect a given Fiber Surface Control Polygon (FSCP) edge by filtering candidates via 2-D intersection tests in range space. Unlike its sibling [Domain-based BVH with Range Bounding Boxes](2808-domain-based-bvh-with-range-bounding-boxes.md), the octree is strictly domain-based, which may result in more returned tetrahedra and higher memory usage, but provides a robust spatial hierarchy for conservative culling. See [Domain-based Octree with Range Bounding Boxes](2806-domain-based-octree-with-range-bounding-boxes.md).

### [Tree64](3486-tree64.md)
Tree64 is a sparse voxel tree data structure with a 4³ (64-way) branching factor, designed to reduce tree depth and per-voxel overhead compared to standard octrees. It achieves higher ray traversal throughput by utilizing compact node layouts and optimized GPU traversal strategies, making it suitable for real-time ray tracing of large voxel scenes. See [Tree64](3486-tree64.md).

#### Dynamic Adaptive Octree Grid (DAOG)
**Definition:** A voxel representation structure that offers significant memory savings compared to Regular Grid and standard Octree, allowing representation of large surfaces with little space [S1, unofficial].
**Memory Usage:** For a 1024x1024x1024 Cornell Box model: Regular Grid uses 1048576 KB, Octree uses 158087 KB, DAOG uses 269 KB. For a 1024x1024x1024 Batman model: Regular Grid uses 1048576 KB, Octree uses 106303 KB, DAOG uses 813 KB [S1, unofficial].
**Construction Time:** For 1024x1024x1024 Cornell Box: Octree takes 5.153s, DAOG takes 67.680s, DAOG serialized takes 0.142ms [S1, unofficial].

#### Tensor Approximation Rendering
**Definition:** Rendering method using tensor approximation combined with a multiresolution octree hierarchy, decompressing the full working set before rendering [S2].
**Steps:** Reconstruction is based on the Tucker formulation. Implement tensor times matrix (TTM) multiplications. GPU Kernel: Each kernel thread decompresses one voxel [S2].
**Formula:** Computational complexity per voxel: $O(M)$, where $M$ is the rank. This grows linearly with $M$, instead of $O(M^3)$ [S2].
**Comparison:** Differs from naive tensor reconstruction by reducing complexity from cubic to linear in rank M via optimized kernel threading [S2].
**Usage:** Proposed by Suter et al. [SIM*11] [S2].

## Background
The concept of using octrees for volumetric data dates back to the early days of computer graphics, with foundational work by Meagher [Mea82] and later adaptations for ray tracing by Williams [Wil81]. The Sparse Voxel Octree (SVO) gained prominence in the 2000s for real-time ray tracing and global illumination, notably through the work of Laine and Karras [LK12] and the integration into production pipelines via OpenVDB by Sony Pictures Imageworks [BSS*11]. The specific variants listed here, such as Tree64 and DAOG, represent specialized optimizations for GPU traversal and memory efficiency, respectively, emerging from research in compressed volume rendering and spatial data structures in the 2010s.

## Sources
[S1] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
[S2] PDF State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
