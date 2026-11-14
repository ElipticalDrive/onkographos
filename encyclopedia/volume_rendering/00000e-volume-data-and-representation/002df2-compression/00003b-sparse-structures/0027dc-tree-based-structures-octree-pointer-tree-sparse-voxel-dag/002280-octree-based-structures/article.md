# Octree-based structures

Also called: Sparse Voxel Octrees (SVO), VDB, Tree64, Implicit Octrees

## Overview
Octree-based structures represent 3D volumetric data using a recursive spatial subdivision where each node has eight children, enabling efficient storage of sparse data by omitting empty regions [S1, unofficial][S2]. They differ from sibling structures like Pointer Trees (variable branching factor) and Sparse Voxel DAGs (subtree merging) by strictly maintaining a fixed 8-way hierarchical topology, which simplifies traversal logic and coordinate mapping at the cost of potentially higher memory overhead for highly redundant geometries compared to DAGs [S1, unofficial][S2].

## Shared principles
All structures in this family share the following mechanism:

1.  **Recursive Subdivision:** The 3D domain is recursively divided into sub-volumes. In standard octrees, each node has 8 children. In pointer trees, the branching factor $N$ varies ($N^3$ children).
2.  **Sparsity Encoding:** Empty regions are represented by null pointers or absence of children, rather than storing zero-valued voxels. This provides compression proportional to the sparsity of the data.
3.  **Hierarchical Traversal:** Rendering or access requires traversing from the root to the leaf. This allows skipping large empty volumes (culling) and adapting resolution to view distance or detail requirements.
4.  **Spatial Locality:** Children of a node are spatially contiguous. This locality is often exploited via Morton ordering (Z-order curve) to map 3D coordinates to 1D keys, enabling efficient neighbor lookups and linear array representations in implicit variants [G].
5.  **Subtree Merging:** The core differentiator of the DAG family is the identification of structurally identical subtrees (isomorphic nodes) and their replacement with a single shared instance. This transforms the tree topology into a graph where multiple parents may reference the same child node, eliminating redundancy [S1, unofficial][S2].

## Subtopics

### [Implicit Octree Storage](939-implicit-octree-storage.md)
Implicit Octree Storage is a serialization technique for sparse voxel octrees that eliminates explicit node pointers by sorting the tree nodes into a linear array using an implicit traversal order, such as breadth-first order. It solves the problem of high memory overhead associated with pointer-based tree structures, making it suitable for streaming or persistent storage of large volumetric datasets. The primary trade-off is that while it minimizes memory footprint, the resulting linear layout is not suitable for efficient random-access traversal.

### [Implicit Octree Traversal](1220-implicit-octree-traversal.md)
Implicit Octree Traversal is a method for extracting isosurfaces from sparse volumetric data by building a sparse octree indexed by Morton coordinates and traversing leaf nodes to invoke callbacks for 2x2x2 cubes of neighboring vertices. It solves the problem of efficiently locating leaf nodes and their adjacent neighbors in a sparse representation without explicitly storing the full cell structure, which would cause storage explosion. It differs from [Pointerless Octree for Seed Placement](5280-pointerless-octree-for-seed-placement.md) by focusing on general point-location and neighbor-finding for surface extraction rather than specific seed placement logic, and from [Cell Retrieval via Neighbor-Finding](1822-cell-retrieval-via-neighbor-finding.md) by providing the underlying hashing primitives for corner value retrieval.

### [OpenVDB (VDB)](1508-openvdb-vdb.md)
OpenVDB is a sparse voxel representation used for the hierarchical encoding and compression of volumetric grids, featuring fast $O(1)$ random access for both modification and retrieval operations. It solves the problem of efficiently storing and accessing vast volumetric datasets where large regions are empty, by storing voxels sparsely so that empty space costs almost nothing on disk or in memory. Unlike dense texture-based approaches, OpenVDB uses a shallow, four-level tree structure with direct-access bitmasks to enable efficient traversal and modification, making it the de facto standard file format for storing volumetric data in production rendering and visual effects.

### [Time-Variant Volume Rendering with Octrees](2232-time-variant-volume-rendering-with-octrees.md)
Time-Variant Volume Rendering with Octrees addresses the limitation of GPU memory and bus bandwidth when rendering sequences of time-variant volume data. By compressing multiple timesteps into a sparse octree representation, the method allows large datasets to reside in limited CPU main memory, enabling interactive rendering of full sequences that would otherwise exceed GPU capacity. It sets itself apart from standard sparse structures by explicitly exploiting temporal coherence to fit entire time-series into memory, whereas alternatives like [Implicit Octree Storage](939-implicit-octree-storage.md) focus on static spatial compression.

### Tree64
Tree64 is a sparse voxel tree structure with a 4³ branching factor, designed to reduce tree depth and memory overhead compared to standard octrees [S1, unofficial][S2]. It achieves higher traversal throughput by skipping 64 voxels per node level, resulting in a tree approximately half as deep as an octree for the same resolution [S2]. It is primarily used for real-time ray tracing of large voxel scenes [S1, unofficial].

### [Sparse Octree Terrain Representation](8349-sparse-octree-terrain-representation.md)
Sparse Octree Terrain Representation replaces dense heightmap diffusion with a sparse octree structure that stores only occupied voxels, aligning with the Voxy schema to avoid wasting compute on empty cells. Unlike previous methods that diffuse all 256 values of a full 16x16 heightmap regardless of terrain presence, this method works only over the occupied 8x8 patch surface, storing only non-empty octree nodes.

### [Voxel Simplification for LOD](9737-voxel-simplification-for-lod.md)
This method approximates triangular models to varying levels of detail (LOD) by converting them into a sparse voxel octree representation of a C0-continuous signed distance field (SDF). It solves the problem of storing high-detail geometry efficiently by using an incremental simplification of function values based on a novel error metric that minimizes surface approximation error. Unlike traditional LOD systems that require pre-generated distinct model versions, this method determines LOD dynamically by viewer distance, representing the surface with necessary details without the computational overhead of loading different assets.

### [Implicit Voxel Simplification with Error Metric](9838-implicit-voxel-simplification-with-error-metric.md)
This method simplifies an implicit voxel model by coarsening the grid hierarchy where the signed distance function is smooth, minimizing the approximation error of the surface geometry. It differs from standard geometric simplification by operating directly on the scalar field values to preserve the zero-level set (the surface) rather than mesh vertices, and it differs from lossy compression by providing a mathematically optimal least-squares fit for the coarse representation.

### [Traversal Acceleration](10779-traversal-acceleration.md)
Traversal Acceleration refers to a family of techniques that utilize hierarchical data structures, primarily octrees, to optimize the access patterns and computational cost of volume rendering and geometric queries. It solves the problem of excessive processing time caused by traversing large, sparse, or complex volumetric datasets by enabling the skipping of empty or irrelevant regions. What sets this family apart from its siblings is the specific application of acceleration structures (such as min-max octrees, summed area tables, or shadow grids) to bypass expensive per-voxel computations during rendering or intersection tests, rather than just storing the data itself.

### [Sparse Voxel Brick Set](11886-sparse-voxel-brick-set.md)
The Sparse Voxel Brick Set is a sparse volume representation that groups Signed Distance Field (SDF) data into fixed-size cubic blocks called "bricks," each associated with a raytracing Axis-Aligned Bounding Box (AABB). It solves the trade-off between the memory efficiency of hierarchical structures and the real-time constructibility of flat grids, specifically optimizing for GPU cache coherency and parallel construction. Unlike Sparse Voxel Octrees (SVOs) which vary resolution, or Grid Sphere Tracing which stores every voxel, the Brick Set uses a uniform brick granularity to balance memory usage and traversal speed, making it particularly effective in complex scenes.

### Dynamic Adaptive Octree Grid (DAOG)
- **Definition:** A voxel representation structure that offers significant memory savings compared to Regular Grid and standard Octree, allowing representation of large surfaces with little space [S1, unofficial].
- **Memory Usage:** For a $1024^3$ Cornell Box model, Regular Grid uses 1048576 KB, Octree uses 158087 KB, and DAOG uses 269 KB [S1, unofficial]. For a Batman model, Regular Grid uses 1048576 KB, Octree uses 106303 KB, and DAOG uses 813 KB [S1, unofficial].
- **Construction Time:** For a $1024^3$ Cornell Box model, Octree construction takes 5.153s, DAOG takes 67.680s, and serialized DAOG takes 0.142ms [S1, unofficial].

### Tensor Approximation Rendering
- **Definition:** Rendering method using tensor approximation combined with a multiresolution octree hierarchy, decompressing the full working set before rendering [S2].
- **Steps:** Reconstruction is based on the Tucker formulation. Implementation requires tensor times matrix (TTM) multiplications. A GPU kernel decompresses one voxel per thread [S2].
- **Complexity:** Computational complexity per voxel is $O(M)$, where $M$ is the rank. This grows linearly with $M$, instead of $O(M^3)$ [S2].
- **Comparison:** Differs from naive tensor reconstruction by reducing complexity from cubic to linear in rank $M$ via optimized kernel threading [S2].
- **Usage:** Proposed by Suter et al. [SIM*11] [S2].

### Vector-based Octree Storage
- **Definition:** Storing octree nodes as `vector3` structures where $x, y, z$ represent the center point and $w$ is a flags field [S3, unofficial].
- **Steps:** Use the $w$ component flags field to denote which of the 8 octant child nodes follow the current node, requiring 8 bits. Store entities as bounding boxes using $r, g, b$ for dimensions and $w$ for other data. Define a special vector (e.g., if $w + z$ equals a magic value) to indicate that an object list follows. Walk the octree and output vectors to a memory buffer [S3, unofficial].
- **Parameter:** A $512 \times 512$ texture can hold a fully packed octree 5 levels deep (32,768 nodes) with 8 objects each, or a 4-level octree with 64 objects each [S3, unofficial].

### Sparse Voxel Octree with 3x3 Bricks
- **Parameter:** Brick size is 3x3 voxels per octree node corner [S4, unofficial].
- **Memory:** Memory usage is less than half that of previous solutions [S4, unofficial].

### Sparse Octree with Interval Evaluation
- **Definition:** An optimization for volume rendering/meshing that uses a sparse octree and interval arithmetic to quickly discard large empty or full regions, reducing the number of cubes evaluated [S5, unofficial].
- **Parameter:** The voxel evaluation bounding volume must be large enough to encompass the entire object, otherwise the output mesh is truncated [S5, unofficial].

### Sparse Spatial Octree
- **Definition:** A pointer-based Sparse Spatial Octree optimized for volumetric data such as entity trails and scent fields, featuring efficient storage via recursive subdivision for fine resolution in active areas while remaining memory-sparse [S6, unofficial].
- **Steps:**
    - **Diffusion:** Spreads density between neighboring voxels.
    - **Decay:** Fades density over time.
    - **Drift:** Simulates uniform environmental motion (e.g., wind) by shifting the octree's world-space reference frame in $O(1)$ time.
    - **Gradient Following:** Enables entities to navigate using the density field's gradient, supporting ant-colony-style behaviors.
    - **Pruning Optimization:** Internal nodes store the maximum density of their children, preventing dilution and ensuring correct hierarchical traversal even for faint trails [S6, unofficial].

## Background
Octrees originated as a spatial partitioning data structure for efficient 3D scene management. In volume rendering, they were adapted to handle sparse voxel data, allowing for the compression of large volumetric datasets where most of the space is empty. OpenVDB, developed by DreamWorks Animation, popularized the shallow, four-level octree structure with bitmasking for visual effects pipelines [S1, unofficial][S2]. The family has expanded to include implicit storage formats for streaming [S1, unofficial], specialized structures like Tree64 for ray tracing [S2], and methods for time-variant data [S1, unofficial][S2].

## Sources
[S1] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
[S2] PDF State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S3] Efficient storage for a sparse octree?, https://stackoverflow.com/questions/6085209/efficient-storage-for-a-sparse-octree, fetched 2026-10-26 (unofficial)
[S4] Interactive Indirect Illumination Using Voxel Cone Tracing: An Insight, https://openalex.org/W4296123939, fetched 2026-10-26 (unofficial)
[S5] 2025-03-24, https://incoherency.co.uk/notes/20250324.html, fetched 2026-10-30 (unofficial)
[S6] Sparse Spatial Octree for Trails and Diffusion, https://github.com/ricecake/boidsish/pull/7, fetched 2026-10-26 (unofficial)
