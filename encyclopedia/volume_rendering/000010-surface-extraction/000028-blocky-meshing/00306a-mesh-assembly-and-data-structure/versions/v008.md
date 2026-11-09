# Mesh Assembly and Data Structure

Also called: Block Mesh Assembly, Chunk Mesh Assembly, Mesh Plugin Architecture, Block Render Data Structure

## Overview
This family manages the transition from abstract voxel topology to GPU-drawable geometry. It solves the problem of fragmented geometry and heterogeneous block types by aggregating mesh data into contiguous typed arrays, defining extensible algorithmic contracts, and handling per-instance metadata such as texture IDs and tints [S1, S2, 4100-chunk-mesh-assembly.md, 4921-mesh-plugin-architecture.md, 6209-block-render-data-structure-instance-generation.md, unofficial]. Unlike [Blocky meshing](31-blocky-meshing.md), which focuses on face culling and greedy merging to reduce quad count, this family handles the buffer layout, material assignment, and data structure definitions required for efficient submission [31-blocky-meshing.md].

## Shared principles
The family operates on a pipeline where raw voxel data is converted into structured mesh primitives, which are then organized into buffers suitable for GPU rendering.

1.  **Data Contiguity**: Mesh data is stored in contiguous typed arrays to minimize memory fragmentation and maximize GPU transfer efficiency. Common attributes include positions (`Float32`), normals (`Int8`), and block identifiers (`Uint16`/`Uint32`) [4100-chunk-mesh-assembly.md, unofficial].
2.  **Layer Separation**: Geometry is often separated into render layers (e.g., Opaque, Water, Transparent) to allow for distinct sorting and shader usage. Each layer maintains its own vertex and index offsets within the global buffer [4100-chunk-mesh-assembly.md, unofficial].
3.  **Material Mapping**: Block types are mapped to materials. If a specific material is missing, a fallback material is used to prevent null dereferences [4100-chunk-mesh-assembly.md, unofficial].
4.  **Extensibility**: The meshing process is decoupled from the assembly process via plugin interfaces, allowing different algorithms (e.g., greedy meshing vs. marching cubes) to produce standardized mesh data structures [4921-mesh-plugin-architecture.md, unofficial].
5.  **Instance Metadata**: Per-block metadata, such as texture IDs and tint colors, is resolved and mixed before being passed to the instance generation phase [6209-block-render-data-structure-instance-generation.md, unofficial].

## Subtopics

### [Chunk Mesh Assembly](4100-chunk-mesh-assembly.md)
Aggregates mesh data from multiple rendering layers into contiguous typed arrays for efficient GPU submission. It maps block types to materials and handles per-vertex attributes like ambient occlusion and light levels. It differs from [Block Render Data Structure & Instance Generation](6209-block-render-data-structure-instance-generation.md) by packing all quads for a chunk into a single mesh object with submesh descriptors rather than relying on instancing or separate draw calls per block type. Choose this when you need a unified buffer structure for a chunk.

### [Mesh Plugin Architecture](4921-mesh-plugin-architecture.md)
Provides an interface for pluggable meshing algorithms, decoupling mesh generation logic from chunk management. It allows the selection of different algorithms (e.g., greedy meshing for blocky voxels vs. marching cubes for smooth terrain) based on voxel type. It differs from [Chunk Mesh Assembly](4100-chunk-mesh-assembly.md) by defining the algorithmic contract rather than the buffer population and submission process. Choose this when you need to support multiple meshing algorithms within the same engine.

### [Block Render Data Structure & Instance Generation](6209-block-render-data-structure-instance-generation.md)
Defines the per-block metadata required to generate mesh instances, focusing on the data layout and tint mixing logic. It resolves texture names to IDs and mixes tints via element-wise multiplication. It differs from sibling methods by focusing on the high-level block definition (textures, tints, shapes) rather than low-level buffer assembly. Choose this when you need to support dynamic texture tinting and complex block shapes.

### Mesh Configuration and Data Structures
*   **Parameter**: Data structures for block IDs must be native JavaScript `Set`. Effect's `HashSet` is forbidden due to performance constraints (~400k membership tests per chunk) [S1, unofficial].
*   **Layout**: BlockId values must be registry-derived. Untrusted numeric inputs must be converted using `BlockId.result` API before building sets/maps [S1, unofficial]. Transparent solids are split into two sets: `transparentSolidBlockIds` (glass, leaves - atlas material + alpha blend) and water (`waterBlockIds` - dedicated shader for ripples/refraction/variable height) [S1, unofficial]. FluidQuad structure includes `direction`, `flow` (optional, contains `direction` vector and `falling` flag), and geometry properties [S1, unofficial]. ChunkView structure holds `coord`, per-chunk `height`, and a byte view for meshing. It does not implicitly truncate or zero-pad different heights [S1, unofficial]. Storage layout (`blockIndex`) is identical to the reference implementation to allow direct use of reference chunk fixtures as golden inputs [S1, unofficial].

### Material-Based Mesh Sectioning
*   **Definition**: Splits a decoded mesh into separate sections (arrays of triangles) based on the material ID associated with each triangle's vertices [S2, unofficial].
*   **Step**: Decode the input mesh using `GetDecodedMesh`. Perform a sanity check: if the number of indices is less than 3, return an empty mesh sections array [S2, unofficial]. Iterate through the decoded mesh indices in steps of 3 to process each triangle [S2, unofficial]. Retrieve the three vertices for the current triangle using the indices. Add the mesh offset to each vertex's position [S2, unofficial]. Determine the material ID for the triangle. Start with Vertex 0's material ID. If Vertex 1 and Vertex 2 have the same material ID, use that instead (implying a preference for consistency or a specific rule for mixed materials) [S2, unofficial]. Ensure the mesh sections array is large enough to hold the material ID. If `materialID` is greater than or equal to the current number of sections, resize the array to `materialID + 1` with zeroed elements [S2, unofficial]. Add the triangle to the section corresponding to its material ID [S2, unofficial].

## Background
The concepts originate from voxel game engines and real-time terrain rendering systems. The specific implementations cited come from community-driven repositories such as `nerima-games/mc-meshing` and `Jay2645/VoxelGenerator`, which adapt Minecraft-style meshing techniques for modern web and game engines [S1, S2, unofficial]. The shift from naive meshing to optimized assembly and plugin architectures reflects the need for performance in large-scale voxel worlds, where memory bandwidth and draw call overhead are critical bottlenecks [S1, S2, unofficial].

## Sources
[S1] nerima-games/mc-meshing, https://github.com/nerima-games/mc-meshing, fetched 2026-10-28 (unofficial)
[S2] Jay2645/VoxelGenerator, https://github.com/Jay2645/VoxelGenerator, fetched 2026-10-27 (unofficial)
