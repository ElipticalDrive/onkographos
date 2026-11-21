# Sparse Voxel Grid


## Overview
A sparse voxel grid is a volume representation that stores only non-zero or active elements as a list of coordinates and their corresponding data, discarding empty space to reduce memory usage [S1, unofficial]. It solves the problem of excessive memory consumption in high-resolution dense grids by leveraging the sparsity of 3D space [S1, unofficial]. It builds on the shared principles of [Sparse Voxel Storage and Chunking](15595-sparse-voxel-storage-and-chunking.md) by explicitly tracking active voxels and mapping integer coordinates to object space.

## How it works
**Inputs:** A set of active voxel coordinates $(x, y, z)$ and their associated data values (e.g., density, color, material index) [S1, unofficial].
**Outputs:** A data structure supporting lookup of data values given a coordinate, or iteration over all active voxels [G].

**Data Structures:**
The core structure is a list of active elements. Each element consists of:
1.  **Coordinate:** Integer tuple $(x, y, z)$ [S1, unofficial].
2.  **Data:** The value associated with the voxel (e.g., float, uint8) [S1, unofficial].

To accelerate querying, the list is organized into an accelerated data structure:
*   **Tree:** A binary tree variant of a KD-tree or an Octree (where each node has 8 children) [S1, unofficial].
*   **Hash Table:** A hash map mapping coordinates to data values [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Create an empty list or hash map [G].
2.  **Insertion:** For each active voxel at $(x, y, z)$ with value $v$:
    *   If using a list: Append $(x, y, z, v)$ to the list.
    *   If using a hash table: Compute hash $h(x, y, z)$ and store $v$ at key $h$ [G].
    *   If using a tree: Traverse or insert into the spatial hierarchy (e.g., Octree) based on coordinate ranges [G].
3.  **Querying (Lookup):** Given a target coordinate $(x_t, y_t, z_t)$:
    *   If using a list: Iterate through all elements. If an element's coordinates match $(x_t, y_t, z_t)$, return its data. If no match is found, return the background value [S1, unofficial].
    *   If using a hash table: Compute $h(x_t, y_t, z_t)$ and retrieve the value. If the key does not exist, return the background value [G].
    *   If using a tree: Traverse the tree to the leaf node corresponding to $(x_t, y_t, z_t)$ and retrieve the data. If the leaf is empty, return the background value [G].

**Formulas:**
*   **Flattened Index:** For grid resolution $n$ and coordinates $(x, y, z)$, the index in the material array is:
    $$ \text{index} = z \times (n+1)^2 + y \times (n+1) + x $$
    [S2]
*   **Ending Point Index:** Given a starting point index $a$, the ending point index $b$ for adjacent points is:
    *   X-axis: $b = a + 1$
    *   Y-axis: $b = a + (n+1)$
    *   Z-axis: $b = a + (n+1)^2$
    [S2]

**Parameters:**
*   **Search Structure:** Choice between list, tree, or hash table. Lists are simple but slow ($O(N)$). Trees and hash tables offer faster lookup ($O(\log N)$ or $O(1)$ average) [S1, unofficial].
*   **Tree Type:** Octree (8 children per node) is a common variant for 3D space [S1, unofficial].
*   **Grid Resolution ($n$):** Integer from 1 to 256 [S2].

**Edge Cases and Pitfalls:**
*   **Querying Performance:** Simple iteration over the list is inefficient for large grids. Accelerated structures are recommended for practical use [S1, unofficial].
*   **Empty Space:** Voxels not in the list are implicitly empty [S1, unofficial].
*   **Precision:** Coordinates are integers. Floating-point positions must be discretized to grid indices before lookup [G].

## Subtopics

### [Point Implicit Contouring (PIC)](16663-point-implicit-contouring-pic.md)
Point Implicit Contouring (PIC) is a sparse voxel storage format that represents a 3D object as a dense voxel grid of material indices or signs, combined with a sparse set of points lying on the object's surface within boundary cells. It derives mesh connectivity implicitly from the voxel data using dual triangulation methods. It is distinguished from its sibling [Ray-Rep](8022-ray-representation.md) by supporting surface smoothing and eliminating the need for post-editing regularization, though it trades off some CSG speed for this flexibility. Choose it when implicit surface reconstruction and smoothing are required over explicit CSG operations. [Link: [Point Implicit Contouring (PIC)](16663-point-implicit-contouring-pic.md)]

### [Voxel Association](19071-voxel-association.md)
Voxel Association is a mechanism for applying spatial regularizers (e.g., Eikonal, smoothness, Laplacian) to multi-resolution sparse voxel structures by reasoning on a single conceptual finest lattice. It solves the problem of spatial coupling in sparse voxel grids, which possess hard boundaries and lack the built-in spatial continuity of implicit neural fields or Gaussian primitives. Unlike hashmap-based approaches that assume a uniform grid, it maintains a conceptual finest grid to enable consistent neighborhood queries across different Levels of Detail (LoD) while storing only occupied cells. Choose it when spatial continuity and multi-resolution regularization are needed. [Link: [Voxel Association](19071-voxel-association.md)]

### Sparse Volume Data Storage
A storage strategy for volumetric terrain that keeps sampled volume data in separate cells, discarding Signed Distance Fields (SDFs) after Hermite data construction to allow parallel modifications and extractions [S2]. It distinguishes itself by storing edge data (edges, zero crossings, normals) sparsely in typed arrays simulating a hash map, while material indices are stored in a dense 1D array [S2]. It is chosen for dynamic terrain engines requiring parallel updates and efficient Web Worker communication [S2].

**Layout:**
*   **Material Indices:** One-dimensional typed array of 8-bit unsigned integers (uint8) [S2].
*   **Edge Data:** Typed arrays simulating a hash map, split into three groups (edges, Zero Crossings, normals), each further split into three arrays for X, Y, Z axes. Edges are stored as starting grid point indices in ascending order [S2].
*   **Zero Crossing Values:** Stored in order corresponding to the edges [S2].
*   **Normal Vectors:** Stored as (x, y, z) floating point triples, corresponding to the order of the edges [S2].

**Formulas:**
*   **Flattened Index:** For grid resolution $n$ and coordinates $(x, y, z)$, the index in the material array is:
    $$ \text{index} = z \times (n+1)^2 + y \times (n+1) + x $$
    [S2]
*   **Ending Point Index:** Given a starting point index $a$, the ending point index $b$ for adjacent points is:
    *   X-axis: $b = a + 1$
    *   Y-axis: $b = a + (n+1)$
    *   Z-axis: $b = a + (n+1)^2$
    [S2]

**Parameters:**
*   **Grid Resolution ($n$):** Integer from 1 to 256 [S2].

### Weight-sharing Voxel Representation
A voxel representation where weights are shared across different spatial scales to reduce memory usage and enhance performance [S3]. It is distinguished from independent multi-scale voxels by reducing model size while maintaining or improving rendering quality [S3]. It is chosen when memory efficiency and training speed are critical, such as in few-shot novel view synthesis [S3].

## Operating characteristics
*   **Time Complexity:**
    *   **List-based:** Lookup is $O(N)$ where $N$ is the number of active voxels [S1, unofficial].
    *   **Tree-based:** Lookup is $O(\log N)$ for balanced trees like Octrees [S1, unofficial].
    *   **Hash-based:** Lookup is $O(1)$ average case [S1, unofficial].
*   **Memory Complexity:** $O(N)$ where $N$ is the number of active voxels, significantly less than $O(R^3)$ for a dense grid of resolution $R$ when sparsity is high [S1, unofficial].
*   **Strengths:** Efficient memory usage for sparse volumes; simple data structure for lists [S1, unofficial].
*   **Limitations:** List-based lookup is slow; hash tables may have high overhead for very small datasets; tree construction can be complex [S1, unofficial].
*   **When to Use:** When the volume is highly sparse and memory is a constraint [S1, unofficial].
*   **When Not to Use:** Not given in the sources.
*   **Measured Performance and Scaling:** Not given in the sources.

## Background
The concept of sparse voxel grids originates from the need to handle high-resolution 3D data efficiently [S1, unofficial]. Chris Choy discusses sparse voxel grids in the context of accelerating Neural Radiance Fields (NeRF) inference by reducing the number of feed-forward passes through empty space [S1, unofficial]. The term "Neural Sparse Voxel Fields" refers to methods that learn such grids progressively [S1, unofficial]. The weight-sharing variant was introduced in FrugalNeRF to improve convergence and reduce memory in few-shot novel view synthesis [S3]. The Sparse Volume Data Storage variant was developed for WebGL-based volumetric terrain engines to enable parallel modifications and efficient data transfer to Web Workers [S2].

## Sources
[S1] Faster Neural Radiance Fields Inference — Chris Choy, https://chrischoy.org/posts/nerfs/, fetched 2026-10-29 (unofficial)
[S2] Volumetric Terrain Rendering with WebGL, https://doi.org/10.2352/issn.2470-1173.2018.06.mobmu-136, fetched 2026-10-26
[S3] FrugalNeRF: Fast Convergence for Extreme Few-shot Novel View Synthesis without Learned Priors, https://doi.org/10.1109/cvpr52734.2025.01049, fetched 2026-10-29
