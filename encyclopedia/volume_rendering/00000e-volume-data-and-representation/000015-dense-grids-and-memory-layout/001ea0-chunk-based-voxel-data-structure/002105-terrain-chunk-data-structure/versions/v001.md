# Terrain Chunk Data Structure

Also called: 4-8 hierarchy, Terrain Representation, TerrainChunkCollection, bintree hierarchy, triangular patches

## Overview
The Terrain Chunk Data Structure represents a height-field-based terrain as a hierarchical collection of triangular patches, enabling efficient multiresolution rendering and compression. Unlike the parent [Chunk-based Voxel Data Structure](1818-chunk-based-voxel-data-structure.md) which manages discrete voxel IDs, this method manages continuous geometric surfaces (triangles) grouped into chunks for batched rendering, cache locality, and adaptive refinement. It is distinct from [Sparse Voxel Octree Representation](2340-sparse-voxel-octree-representation.md) by focusing on surface meshes rather than volumetric occupancy, and from [Compact Voxel Section Storage](4219-compact-voxel-section-storage.md) by using a bintree hierarchy of patches instead of fixed 3D grids.

## How it works
**Inputs and Outputs:**
*   **Input:** A terrain height field (implicit or explicit) and a set of refinement criteria (e.g., object-space error tolerance, view distance).
*   **Output:** A hierarchy of triangular patches organized in a 4-8 bintree, where each patch contains hundreds of triangles.

**Data Structures:**
The structure consists of three primary components defined in [S2, unofficial]:

1.  **TerrainChunk_Val (Value Type):** Stores the immutable state of a chunk.
    *   `ChunkIndex`: `int3` (32-bit integer vector).
    *   `MinCornerWS`: `float3` (World Space minimum corner).
    *   `ChunkSize`: `float3`.
    *   `ChunkLOD`: `TerrainLod` (enumeration or integer representing Level of Detail).

2.  **TerrainChunk_Ref (Reference Type):** Extends the value type with computed geometry bounds.
    *   `ChunkIndex`: `int3`.
    *   `MinCornerWS`: `float3`.
    *   `ChunkSize`: `float3`.
    *   `MaxCornerWS`: `float3` (Computed as `MinCornerWS + ChunkSize`).
    *   `ChunkLOD`: `TerrainLod`.

3.  **TerrainChunkCollection (Manager):** Manages the array of chunks.
    *   `TerrainChunks`: Array of `TerrainChunk_Val`.
    *   `Resolution`: `int3` (Total world resolution in chunks or voxels).
    *   `ChunkDimension`: `int3` (Dimensions of a single chunk).
    *   `Origin`: `float3` (World space origin of the terrain).

**Hierarchy and Indexing:**
The terrain is organized as an adaptive bintree (4-8 hierarchy) [S1]. This structure allows for constant-time computation of indices for children, parents, siblings, and cousins that form a "diamond" pattern (e.g., indices 13 and 14) [S1]. The hierarchy supports two connectivity modes for refinement:
1.  **4-8 Connectivity:** Adaptive refinement where a patch can split into 4 or 8 children depending on the geometry and error metrics.
2.  **Regular Connectivity:** Uniform subdivision.

**Algorithm Steps:**
1.  **Initialization:** Create a `TerrainChunkCollection` with the global `Resolution`, `ChunkDimension`, and `Origin`. Initialize the root level of the bintree with coarse triangular patches.
2.  **Refinement:** Traverse the hierarchy. For each patch, calculate the approximation error against the underlying height field.
    *   If the error exceeds the threshold, refine the patch using 4-8 connectivity [S1].
    *   If the error is below the threshold and the patch is not at the minimum LOD, coarsen it.
3.  **Mesh Generation:** For each leaf patch in the hierarchy, generate a mesh of hundreds of triangles [S1]. The vertices are sampled from the height field.
4.  **Culling and Paging:** Use the `MinCornerWS` and `MaxCornerWS` from `TerrainChunk_Ref` for view frustum culling. Manage memory by paging compressed patches between external storage, main memory, and GPU memory [S1].
5.  **Rendering:** Batch render the triangles from the active patches. The grouping into patches reduces mesh adaptation costs and supports efficient batched rendering [S1].

**Formulas:**
*   **Max Corner Computation:**
    $$ \text{MaxCornerWS} = \text{MinCornerWS} + \text{ChunkSize} $$
*   **Diamond Indexing:** The indexing scheme allows constant time access to neighbors. If $I$ is the index of a node, its children, parent, and diamond neighbors (e.g., cousins) are computed via bitwise operations or arithmetic offsets defined by the bintree layout [S1]. Specific bit-packing details for the index are not given in the sources.

**Parameters:**
*   **Error Tolerance:** Object-space error tolerance used for adaptive resampling [S1].
*   **Precision:** Quantization to 12 bits for z-elevations and 10 bits for xy coordinates is mentioned in related work (Dick et al.) but not explicitly as a parameter of this specific structure's definition [S1].

**Edge Cases and Pitfalls:**
*   **Cracks:** The multiresolution hierarchy must be managed to avoid cracks between patches of different LODs. The sources mention "without cracks" as a requirement for the compression scheme [S1].
*   **Precision:** Lossy coding of lower resolution data may introduce artifacts if precision is not managed [S1].

## Operating characteristics
*   **Time Complexity:** Constant time for index computation of hierarchical neighbors (children, parents, siblings, diamond cousins) [S1]. Refinement cost depends on the number of patches and the error calculation complexity.
*   **Memory Complexity:** Depends on the adaptive nature of the 4-8 hierarchy. High-frequency details require more patches. The structure supports compression, reducing memory footprint [S1].
*   **Performance:** Designed for high-speed lossless compression and on-the-fly decompression on the GPU [S1]. Batched rendering of patches reduces draw call overhead.
*   **Strengths:** Fine-grained control over triangle count per patch, efficient caching, spatial locality for compression, and support for view frustum culling [S1].
*   **Limitations:** Requires complex mesh adaptation logic. Not suitable for volumetric data (unlike the parent chunk structure).

## Background
The terrain chunk data structure originates from research on high-speed lossless compression and on-the-fly decompression of multiresolution terrain [S1]. It was developed to address the limitations of previous methods by treating the height field as a texture and using an adaptive bintree hierarchy of triangular patches. The approach allows for efficient batched rendering and cache management, with compressed data fitting on the GPU [S1]. The specific implementation details of `TerrainChunk_Val`, `TerrainChunk_Ref`, and `TerrainChunkCollection` are found in an unofficial repository [S2, unofficial].

## Sources
[S1] On-the-fly decompression and rendering of multiresolution terrain, https://doi.org/10.1145/1730804.1730815, fetched 2026-11-01
[S2] 2JacT/Marching-Cubes-Terrain, https://github.com/2JacT/Marching-Cubes-Terrain, fetched 2026-10-27 (unofficial)
