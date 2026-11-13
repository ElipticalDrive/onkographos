# Mesh Assembly and Data Structure

Also called: Chunk Mesh Assembly, Mesh Plugin Architecture, Block Render Data Structure

## Overview
Mesh Assembly and Data Structure is the family of methods responsible for converting raw voxel geometry into GPU-draw-ready buffer structures. It solves the problem of fragmented geometry and decoupled material definitions by aggregating mesh data from multiple rendering layers into contiguous typed arrays and mapping block types to materials [S2, 4100-chunk-mesh-assembly.md, unofficial]. Unlike [Block Render Data Structure & Instance Generation](6209-block-render-data-structure-instance-generation.md), which focuses on per-block metadata and instance generation, this family packs all quads for a chunk into single mesh objects with submesh descriptors [S2, unofficial]. It differs from [Mesh Plugin Architecture](4921-mesh-plugin-architecture.md) by handling the concrete buffer allocation and assignment rather than defining the algorithmic interface.

## Shared principles
The family shares a common contract for buffer layout and assembly logic. All child methods produce or consume buffers compatible with the following structure [S2, unofficial]:

*   **Positions:** $N \times 4 \times 3$ floats (4 vertices per quad, 3 components).
*   **Normals:** $N \times 4 \times 3$ signed bytes (int8).
*   **Indices:** $N \times 6$ unsigned integers (2 triangles per quad).
*   **Attributes:** $N \times 4$ elements per attribute (Block ID, Ambient Occlusion, Lighting) [S2, unofficial].

The assembly process generally follows these steps:
1.  **Change Detection:** Check if the chunk has modified voxel data. If false, skip assembly [S1, unofficial].
2.  **Quad Count Calculation:** Calculate the total number of quads across all defined render layers [S2, unofficial].
3.  **Buffer Allocation:** Allocate output arrays based on the total quad count [S2, unofficial]:
    *   Positions/Normals size: `quadCount * 4 * 3`.
    *   Indices size: `quadCount * 6`.
    *   Other attributes: `quadCount * 4` per attribute.
4.  **Layer Iteration:** Iterate over each layer in `MESH_BUFFER_LAYERS` [S2, unofficial]:
    *   Record current offsets.
    *   Append quads for the current layer. For each quad, compute vertex positions from bounds, write vertex attributes, and write 6 indices [S2, unofficial].
    *   Look up the material for the block type. If not found, use a pre-scanned `FallbackMat` [S3, unofficial].
    *   Record new offsets in `MeshBufferGroup`.
5.  **Mesh Configuration:** Configure the target mesh object with vertex/index buffers and submesh descriptors [S1, unofficial].
6.  **Assignment:** Assign the mesh to the chunk's `MeshFilter` and `MeshCollider` [S1, unofficial].

## Subtopics

### [Chunk Mesh Assembly](4100-chunk-mesh-assembly.md)
Chunk Mesh Assembly aggregates mesh data from multiple rendering layers (e.g., opaque, water, transparent solid, fluids) into contiguous typed arrays for efficient GPU submission. It solves the problem of fragmented geometry by mapping block types to materials and handling per-vertex attributes such as ambient occlusion and light levels within a unified buffer structure. Unlike [Block Render Data Structure & Instance Generation](6209-block-render-data-structure-instance-generation.md), which may use instancing or separate draw calls per block type, this method packs all quads for a chunk into a single mesh object with submesh descriptors.

### [Mesh Plugin Architecture](4921-mesh-plugin-architecture.md)
The Mesh Plugin Architecture decouples meshing algorithms from the chunk assembly pipeline by defining a standard interface (`IMeshingJob`) for pluggable surface extraction methods. It allows the engine to switch between blocky face-culling (e.g., Greedy Meshing) and smooth terrain generation (e.g., Marching Cubes) without modifying the assembly logic, ensuring consistent output formats for the downstream buffer allocator.

### [Block Render Data Structure & Instance Generation](6209-block-render-data-structure-instance-generation.md)
The Block Render Data Structure & Instance Generation method defines the per-block metadata required to render a voxel face and the process of converting that metadata into GPU draw calls. It solves the problem of decoupling block definition (texture and color) from mesh assembly by providing a structured container for face-specific attributes. It differs from sibling methods by focusing on the *data* passed to the rendering primitive rather than the *geometry* generation or buffer assembly itself.

### Mesh Configuration and Data Structures
*   **Data structures for block IDs must be native JavaScript `Set`.** Effect's `HashSet` is forbidden due to performance constraints (~400k membership tests per chunk) [S1, unofficial].
*   **BlockId values must be registry-derived.** Untrusted numeric inputs must be converted using `BlockId.result` API before building sets/maps [S1, unofficial].
*   **Transparent solids are split into two sets:** `transparentSolidBlockIds` (glass, leaves - atlas material + alpha blend) and water (`waterBlockIds` - dedicated shader for ripples/refraction/variable height) [S1, unofficial].
*   **FluidQuad structure** includes `direction`, `flow` (optional, contains `direction` vector and `falling` flag), and geometry properties [S1, unofficial].
*   **ChunkView structure** holds `coord`, per-chunk `height`, and a byte view for meshing. It does not implicitly truncate or zero-pad different heights [S1, unofficial].
*   **Storage layout (`blockIndex`)** is identical to the reference implementation to allow direct use of reference chunk fixtures as golden inputs [S1, unofficial].

### Material-Based Mesh Sectioning
*   **Definition:** Splits a decoded mesh into separate sections (arrays of triangles) based on the material ID associated with each triangle's vertices [S2, unofficial].
*   **Step 1:** Decode the input mesh using `GetDecodedMesh` [S2, unofficial].
*   **Step 2:** Perform a sanity check: if the number of indices is less than 3, return an empty mesh sections array [S2, unofficial].
*   **Step 3:** Iterate through the decoded mesh indices in steps of 3 to process each triangle [S2, unofficial].
*   **Step 4:** Retrieve the three vertices for the current triangle using the indices. Add the mesh offset to each vertex's position [S2, unofficial].
*   **Step 5:** Determine the material ID for the triangle. Start with Vertex 0's material ID. If Vertex 1 and Vertex 2 have the same material ID, use that instead (implying a preference for consistency or a specific rule for mixed materials) [S2, unofficial].
*   **Step 6:** Ensure the mesh sections array is large enough to hold the material ID. If `materialID` is greater than or equal to the current number of sections, resize the array to `materialID + 1` with zeroed elements [S2, unofficial].
*   **Step 7:** Add the triangle to the section corresponding to its material ID [S2, unofficial].

## Background
The specific implementations described in this family originate from unofficial community projects. The `Chunk Mesh Assembly` and `Material-Based Mesh Sectioning` details are derived from [Jay2645/VoxelGenerator](https://github.com/Jay2645/VoxelGenerator) [S2, unofficial]. The `Mesh Plugin Architecture`, `Mesh Configuration`, and performance constraints (such as the use of native `Set` over `HashSet`) are derived from [nerima-games/mc-meshing](https://github.com/nerima-games/mc-meshing) [S1, unofficial]. The `Block Render Data Structure` is also sourced from the latter [S1, unofficial].

## Sources
[S1] nerima-games/mc-meshing, https://github.com/nerima-games/mc-meshing, fetched 2026-10-28 (unofficial)
[S2] Jay2645/VoxelGenerator, https://github.com/Jay2645/VoxelGenerator, fetched 2026-10-27 (unofficial)
