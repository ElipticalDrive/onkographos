# Octree-based structures

Also called: Sparse Voxel Octrees, SVOs, VDB (specifically the OpenVDB variant)

## Overview
Octree-based structures represent volumetric data by recursively subdividing the 3D domain into a tree of nodes, where empty or homogeneous regions are pruned to achieve sparsity [S1, unofficial]. This family solves the memory and computational inefficiency of dense voxel grids by allocating resources only to occupied space, enabling the storage and rendering of large, complex volumetric datasets on commodity hardware [S1, unofficial][S2]. It sets itself apart from sibling methods like Pointer Trees by standardizing the 8-way branching factor, and from Sparse Voxel DAGs by typically lacking the global deduplication of identical subtrees, though variants like Tree64 and OpenVDB introduce specific optimizations for traversal speed and random access respectively [S1, unofficial][S2].

## Shared principles
All structures in this family share the following mechanism:

1.  **Recursive Subdivision:** The 3D domain is recursively divided into sub-volumes. In standard octrees, each node has 8 children. In pointer trees, the branching factor $N$ varies ($N^3$ children).
2.  **Sparsity Encoding:** Empty regions are represented by null pointers or absence of children, rather than storing zero-valued voxels. This provides compression proportional to the sparsity of the data.
3.  **Hierarchical Traversal:** Rendering or access requires traversing from the root to the leaf. This allows skipping large empty volumes (culling) and adapting resolution to view distance or detail requirements.
4.  **Spatial Locality:** Children of a node are spatially contiguous. This locality is often exploited via Morton ordering (Z-order curve) to map 3D coordinates to 1D keys, enabling efficient neighbor lookups and linear array representations in implicit variants [G].
5.  **Subtree Merging:** The core differentiator of the DAG family is the identification of structurally identical subtrees (isomorphic nodes) and their replacement with a single shared instance. This transforms the tree topology into a graph where multiple parents may reference the same child node, eliminating redundancy [S1, unofficial][S2].

## Subtopics

### [Implicit Octree Storage](939-implicit-octree-storage.md)
Implicit Octree Storage is a memory-efficient serialization technique for sparse voxel octrees that eliminates explicit node pointers by sorting the tree nodes into a linear array using an implicit traversal order, such as breadth-first order. It solves the problem of high memory overhead associated with pointer-based tree structures, making it suitable for streaming or persistent storage of large volumetric datasets. Unlike [Implicit Octree Traversal](1220-implicit-octree-traversal.md), which focuses on GPU-side traversal algorithms, this method focuses on the data layout for storage, and unlike [OpenVDB (VDB)](1508-openvdb-vdb.md), which uses a hybrid structure with explicit pointers for interior nodes, it stores the entire hierarchy implicitly. The primary trade-off is that while it minimizes memory footprint, the resulting linear layout is not suitable for efficient random-access traversal without reconstruction.

### [Implicit Octree Traversal](1220-implicit-octree-traversal.md)
Implicit Octree Traversal is a method for building and traversing a sparse octree representation of a distance field to extract isosurfaces. It samples the distance field to find surface crossings, places vertices at leaf centers, and traverses leaf nodes to invoke a callback for each 2x2x2 cube of neighboring leaf vertices. It sets itself apart from sibling methods by focusing on direct Morton coordinate mapping for neighbor retrieval rather than geometric neighbor-finding or seed placement techniques [G].

### [OpenVDB (VDB)](1508-openvdb-vdb.md)
OpenVDB (VDB) is a sparse voxel representation used for the hierarchical encoding and compression of volumetric grids, featuring fast $O(1)$ random access for both modification and retrieval operations. It solves the problem of storing vast volumetric datasets with large empty regions by encoding sparsity hierarchically, ensuring that empty space costs almost nothing on disk or in memory. Unlike dense texture-based approaches, VDB uses a shallow tree structure with direct-access bitmasks to enable efficient out-of-core streaming and runtime modification, making it the de facto standard file format for storing volumetric data in production rendering and visual effects.

### [Time-Variant Volume Rendering with Octrees](2232-time-variant-volume-rendering-with-octrees.md)
This method enables the interactive rendering of time-variant volume datasets by storing multiple timesteps in limited CPU main memory using octree compression, thereby overcoming GPU memory capacity and bus bandwidth limitations. It sets itself apart from standard sparse structures by allowing full sequences of medium-sized time-variant data to reside in main memory for interactive exploration. Unlike baseline renderers that process full uncompressed volumes, this approach permits residency of large datasets on commodity hardware.

### Tree64
Tree64 is a sparse voxel tree with a 4³ branching factor, designed to reduce tree depth and per-voxel overhead compared to standard octrees [S1, unofficial]. It achieves higher ray traversal throughput by utilizing compact 12-byte node layouts and optimized GPU traversal kernels, making it suitable for real-time rendering of large, sparse voxel scenes [S1, unofficial].

### [Sparse Octree Terrain Representation](8349-sparse-octree-terrain-representation.md)
Sparse Octree Terrain Representation replaces dense heightmap diffusion with a sparse octree structure that stores only occupied voxels, aligning with the Voxy schema to avoid wasting compute on empty cells. It differs from its siblings by operating on a fixed $8 \times 8$ local patch with quantized Y-axis levels rather than the full $16 \times 16$ diffuse grid used in previous pipelines, ensuring that traversal and rendering work only over the actual terrain surface.

### [Voxel Simplification for LOD](9737-voxel-simplification-for-lod.md)
A method to approximate a triangular model to varying levels of detail using an incremental simplification of function values based on voxel grids. It decomposes function values hierarchically as a multi-resolution function, minimizing simplification error in the surface itself. Unlike methods requiring different model versions for LOD, this method determines LOD by viewer distance, representing surface with necessary details without computational overhead of loading different models.

### [Implicit Voxel Simplification with Error Metric](9838-implicit-voxel-simplification-with-error-metric.md)
This method simplifies implicit voxel representations by coarsening a fine grid into a hierarchical coarse grid, minimizing the approximation error of the signed distance field (SDF) using a novel integral-based error metric. It differs from standard geometric simplification by operating directly on the scalar field values to preserve the zero-level set (surface) implicitly, rather than manipulating mesh vertices or explicit voxel occupancy. It builds on [Octree-based structures](2792-octree-based-structures.md) by providing the mathematical foundation for determining which octree nodes can be merged based on field smoothness.

### [Traversal Acceleration](10779-traversal-acceleration.md)
Traversal Acceleration refers to a family of techniques that use hierarchical data structures, primarily octrees, to skip empty or homogeneous regions of a volume during rendering or data extraction, thereby reducing computational cost. It solves the problem of inefficiently processing large, sparse, or complex volumetric datasets by avoiding redundant calculations in areas that do not contribute to the final image or result. Unlike sibling methods that may rely on implicit storage or specific terrain representations, this family focuses on the traversal logic and acceleration structures (like min-max octrees or range bounding boxes) to filter candidates or skip steps.

### Dynamic Adaptive Octree Grid (DAOG)
- **Definition:** A voxel representation structure that offers significant memory savings compared to Regular Grid and standard Octree, allowing representation of large surfaces with little space [S1, unofficial].
- **Memory Usage:** For a $1024 \times 1024 \times 1024$ Cornell Box model: Regular Grid uses 1048576 KB, Octree uses 158087 KB, DAOG uses 269 KB. For a Batman model: Regular Grid uses 1048576 KB, Octree uses 106303 KB, DAOG uses 813 KB [S1, unofficial].
- **Construction Time:** For a $1024 \times 1024 \times 1024$ Cornell Box: Octree takes 5.153s, DAOG takes 67.680s, DAOG serialized takes 0.142ms [S1, unofficial].

### Tensor Approximation Rendering
- **Definition:** Rendering method using tensor approximation combined with a multiresolution octree hierarchy, decompressing the full working set before rendering [S2].
- **Steps:** Reconstruction is based on the Tucker formulation. Implement tensor times matrix (TTM) multiplications. GPU Kernel: Each kernel thread decompresses one voxel [S2].
- **Formula:** Computational complexity per voxel: $O(M)$, where $M$ is the rank. This grows linearly with $M$, instead of $O(M^3)$ [S2].
- **Comparison:** Differs from naive tensor reconstruction by reducing complexity from cubic to linear in rank $M$ via optimized kernel threading [S2].
- **Usage:** Proposed by Suter et al. [SIM*11] [S2].

### Vector-based Octree Storage
- **Definition:** Storing octree nodes as vector3 structures where x, y, z represent the center point and w is a flags field [S3, unofficial].
- **Steps:** Use the w component flags field to denote which of the 8 octant child nodes follow the current node, requiring 8 bits. Store entities as bounding boxes using r, g, b for dimensions and w for other data. Define a special vector (e.g., if w + z equals a magic value) to indicate that an object list follows. Walk the octree and output vectors to a memory buffer [S3, unofficial].
- **Parameter:** A $512 \times 512$ texture can hold a fully packed octree 5 levels deep (32,768 nodes) with 8 objects each, or a 4-level octree with 64 objects each [S3, unofficial].

## Background
The concept of sparse voxel octrees emerged as a solution to the memory constraints of dense volumetric rendering. Early work focused on basic octree traversal for ray casting. The OpenVDB format, developed by DreamWorks Animation, became an industry standard for visual effects due to its shallow tree depth and efficient random access capabilities [S1, unofficial][S2]. The DAOG (Dynamic Adaptive Octree Grid) was introduced to further reduce memory footprint for large-scale surface representations, achieving significant compression ratios over standard octrees [S1, unofficial]. Tensor approximation methods were later integrated to handle high-dimensional data compression within the octree hierarchy, allowing for efficient GPU-based decompression [S2].

## Sources
[S1] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
[S2] PDF State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S3] Efficient storage for a sparse octree?, https://stackoverflow.com/questions/6085209/efficient-storage-for-a-sparse-octree, fetched 2026-10-26 (unofficial)
