# Terrain Chunk Data Structure

Also called: 4-8 hierarchy, Terrain Representation, TerrainChunkCollection, bintree hierarchy, triangular patches

## Overview
The Terrain Chunk Data Structure represents terrain as a 4-8 hierarchy of triangular patches, where each patch contains hundreds of triangles [S1]. This grouping enables efficient batched rendering, reduces mesh adaptation cost, supports view frustum culling, serves as a paging unit, and imposes spatial locality for compression [S1]. Unlike the parent [Chunk-based Voxel Data Structure](1818-chunk-based-voxel-data-structure.md) which manages discrete voxel IDs, this method manages continuous geometric surfaces grouped into chunks [S1][S2, unofficial]. It is distinct from [Sparse Voxel Octree Representation](2340-sparse-voxel-octree-representation.md) by focusing on surface meshes rather than volumetric occupancy, and from [Compact Voxel Section Storage](4219-compact-voxel-section-storage.md) by using a bintree hierarchy of patches instead of fixed 3D grids [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A terrain height field and refinement criteria (e.g., object-space error tolerance) [S1].
*   **Output:** A hierarchy of triangular patches organized in a 4-8 bintree, where each patch contains hundreds of triangles [S1].

**Data Structures:**
The structure consists of three primary components defined in [S2, unofficial]:
1.  **TerrainChunk_Val:** `ChunkIndex` (`int3`), `MinCornerWS` (`float3`), `ChunkSize` (`float3`), `ChunkLOD` (`TerrainLod`) [S2, unofficial].
2.  **TerrainChunk_Ref:** Extends `TerrainChunk_Val` with `MaxCornerWS` (`float3`, computed) [S2, unofficial].
3.  **TerrainChunkCollection:** `TerrainChunks` (`TerrainChunk_Val` array), `Resolution` (`int3`), `ChunkDimension` (`int3`), `Origin` (`float3`) [S2, unofficial].

**Hierarchy and Indexing:**
The terrain is organized as an adaptive bintree [S1]. The indexing scheme allows constant-time computation of indices for children, parents, siblings, and cousins that form a "diamond" pattern (e.g., indices 13 and 14) [S1]. Refinement uses 4-8 connectivity or regular connectivity [S1].

**Algorithm Steps:**
1.  **Initialization:** Create a `TerrainChunkCollection` with global `Resolution`, `ChunkDimension`, and `Origin` [S2, unofficial].
2.  **Traversal & Error Evaluation:** Traverse the bintree hierarchy to evaluate approximation error against the underlying height field using object-space error tolerances [S1].
3.  **Adaptive Refinement:** If the error exceeds the tolerance, refine the patch using 4-8 connectivity or adaptive refinement [S1]. If the error is below the tolerance and the patch is not at minimum LOD, coarsen it [S1].
4.  **Mesh Generation:** For each leaf patch, generate a mesh of hundreds of triangles by sampling vertices from the height field [S1].
5.  **Culling & Paging:** Use `MinCornerWS` and `MaxCornerWS` from `TerrainChunk_Ref` for view frustum culling [S2, unofficial]. Page compressed patches between external storage, main memory, and GPU memory [S1].
6.  **Rendering:** Batch render the triangles from active patches [S1].

**Formulas:**
*   **Max Corner Computation:**
    $$ \text{MaxCornerWS} = \text{MinCornerWS} + \text{ChunkSize} $$ [G]
*   **Diamond Indexing:** Children, parents, and diamond neighbors are computed in constant time via the bintree layout [S1].

**Parameters:**
*   **Error Tolerance:** Object-space error tolerance for adaptive resampling [S1].
*   **Precision:** Quantization to 12 bits for z-elevations and 10 bits for xy coordinates (from related Dick et al. work) [S1].

**Edge Cases and Pitfalls:**
*   **Cracks:** The multiresolution hierarchy must be managed to avoid cracks between patches of different LODs [S1].
*   **Precision:** Lossy coding of lower resolution data may introduce artifacts if precision is not managed [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class TerrainChunk_Val:
    ChunkIndex: int
    MinCornerWS: tuple[float, float, float]
    ChunkSize: tuple[float, float, float]
    ChunkLOD: int

class TerrainChunk_Ref(TerrainChunk_Val):
    def __init__(self, val: TerrainChunk_Val):
        self.ChunkIndex = val.ChunkIndex
        self.MinCornerWS = val.MinCornerWS
        self.ChunkSize = val.ChunkSize
        self.ChunkLOD = val.ChunkLOD
        self.MaxCornerWS = tuple(a + b for a, b in zip(val.MinCornerWS, val.ChunkSize))

class TerrainChunkCollection:
    def __init__(self, resolution: tuple[int, int, int], chunk_dim: tuple[int, int, int], origin: tuple[float, float, float]):
        self.TerrainChunks: list[TerrainChunk_Val] = []
        self.Resolution = resolution
        self.ChunkDimension = chunk_dim
        self.Origin = origin

def refine_patches(collection: TerrainChunkCollection, error_tolerance: float):
    for chunk in collection.TerrainChunks:
        # [G] Sources do not specify error computation or split logic
        error = 0.0
        if error > error_tolerance:
            # [G] Sources do not specify split logic
            pass
        else:
            # [G] Sources do not specify coarsen logic
            pass
```

## Subtopics

### Chunk Data Structure
Represents a cubic chunk as a 3D array of `Option<MappedBlockID>` [S3, unofficial]. The chunk size is fixed at 32 units [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Constant time for hierarchical neighbor index computation [S1]. Refinement cost scales with patch count and error calculation complexity [S1].
*   **Memory Complexity:** Scales with adaptive patch density; compression reduces the overall footprint [S1].
*   **Performance:** Optimized for high-speed lossless compression and on-the-fly decompression, with compressed data fitting on the GPU [S1]. Batched rendering reduces draw call overhead [S1].
*   **Strengths:** Fine-grained triangle count control, efficient caching, spatial locality for compression, and view frustum culling support [S1].
*   **Limitations:** Requires complex mesh adaptation logic; not designed for volumetric data [S1].

## Background
The terrain chunk data structure originates from research on high-speed lossless compression and on-the-fly decompression of multiresolution terrain [S1]. It was developed to treat height fields as textures and use an adaptive bintree hierarchy for efficient batched rendering and cache management [S1]. Implementation details for chunk management are documented in an unofficial repository [S2, unofficial].

## Sources
[S1] On-the-fly decompression and rendering of multiresolution terrain, https://doi.org/10.1145/1730804.1730815, fetched 2026-11-01
[S2] 2JacT/Marching-Cubes-Terrain, https://github.com/2JacT/Marching-Cubes-Terrain, fetched 2026-10-27 (unofficial)
[S3] jameshiew/infinigen, https://github.com/jameshiew/infinigen, fetched 2026-10-27 (unofficial)
