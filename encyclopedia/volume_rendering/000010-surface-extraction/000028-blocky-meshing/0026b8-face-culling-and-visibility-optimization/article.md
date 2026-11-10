# Face Culling and Visibility Optimization

Also called: Interior Polygon Removal, Removing Interior Polygons

## Overview
Face Culling and Visibility Optimization reduces rendering load by reconstructing cubes from triangles and removing sides that are adjacent to solid neighbors, thereby avoiding the rendering of 12 triangles (6 sides) when only 2 triangles (1 side) are visible [S1, unofficial]. It sets itself apart from its siblings by operating on a per-voxel neighbor check basis during mesh creation, rather than using coverage masks, block visibility preprocessing, or opacity thresholds [8735-face-culling.md].

## Shared principles
The core mechanism relies on checking the occupancy of the six immediate neighbors of a voxel at position $P$ with normal $N$. The neighbor position is calculated as:
$$ P_{neighbor} = P + N $$
A face is considered visible (and thus generated) if the neighbor at $P_{neighbor}$ is empty or transparent (depending on the specific culling mode). If the neighbor is solid, the face is internal and is culled [S1, unofficial].

In a naive implementation, this involves iterating through all voxels and their 6 faces, checking for neighbor existence in a set of occupied positions, and adding vertices only if `renderFlag` is true [8735-face-culling.md].

## Subtopics

### [Voxel Block Shape & Culling Configuration](2176-voxel-block-shape-culling-configuration.md)
Defines per-block metadata and geometry rules for heterogeneous block types (opaque, transparent, fluid), decoupling block identity from mesh generation logic.

### [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md)
Removes hidden faces of partial voxel shapes by representing face coverage as a bit mask, operating at sub-face granularity.

### [ChunkMesh Vertex Packing and Management](3566-chunkmesh-vertex-packing-and-management.md)
Combines face culling with compact vertex storage and efficient incremental buffer updates for real-time terrain.

### [Voxel Backface Culling and Internal Face Masking](4569-voxel-backface-culling-and-internal-face-masking.md)
Rejects faces occluded by adjacent voxels or oriented away from the camera using projection space tests and bitmasks.

### [Block Visibility Culling](5334-block-visibility-culling.md)
Determines which voxel blocks require mesh updates based on camera position and rotation, skipping index updates if the view has not changed significantly.

### [Extraction of Surface Points (Sobierajski)](5721-extraction-of-surface-points-sobierajski.md)
Extracts only visible boundary voxels from volume data using orthographic views, generating a pre-filtered list of surface points.

### [Mask Projection Culling](6836-mask-projection-culling.md)
Removes faces and vertices not consistently observed across multiple camera poses by projecting points into camera space and comparing depth against rendered depth maps.

### [Mesh Culling via 3D Projection and Connected Components](6963-mesh-culling-via-3d-projection-and-connected-components.md)
Removes geometric artifacts and occluded geometry by projecting vertices into multiple camera views and comparing depths against reference depth maps.

### [Face Culling](8735-face-culling.md)
Reduces vertices sent to the GPU by skipping rendering of voxel faces adjacent to existing neighboring voxels.

### [Opacity-based Mesh Culling](8759-opacity-based-mesh-culling.md)
Post-processing technique that discards faces contributing negligible visual information based on opacity values.

### [From-Region Visibility Preprocessing](9048-from-region-visibility-preprocessing.md)
Precomputes visibility for specific viewpoints or regions to cull occluded geometry during rendering.

### Buried Bitmask Optimization
A bitmask used to track which blocks in a chunk are 'buried' (fully surrounded by opaque blocks), allowing the renderer to skip mesh generation for those blocks [S2, unofficial]. The process resets the buried bitmask for the chunk, creates `MappedAdjacentChunks` to access neighbors, and iterates through all blocks to determine if they are buried via `block->isBuried(...)`. If all blocks are buried, the mesh is marked empty. During mesh generation, any block that is air or has its buried flag set is skipped [S2, unofficial].

### Two-pass transparent meshing with opaque separation
A meshing strategy that separates opaque and transparent geometry into two passes to ensure correct rendering order and reduce overdraw [S3, unofficial]. First, it checks if the chunk contains transparent blocks; if not, the second pass is skipped. In the first (opaque) pass, if transparent blocks exist, it tests for opaque blocks (ignoring transparent and air) to generate opaque faces between air and transparent blocks; otherwise, it tests for non-air blocks. In the second (transparent) pass, it tests for non-air blocks to mark only transparent blocks, generating faces for transparent blocks touching air but not adding faces between transparent and solid blocks. Solid block types are filtered out during the transparent pass by checking if the block is transparent before constructing AO values [S3, unofficial].

### Smart Leaf Rendering
A leaf block rendering strategy that preserves alpha cutouts on outer leaf surfaces while culling internal faces to reduce vertex count [S4, unofficial]. It is controlled by a global static enum `sLeafRenderMode`, which defaults to `ChunkRenderer::LeafRenderMode::Smart` [S4, unofficial].

## Background
The technique originated as an optimization to replace the brute-force approach of instantiating models of cubes, which led to unnecessary rendering of internal triangles [S1, unofficial]. A class was added to reconstruct cubes from triangles, allowing each cube to check for solid neighbors and determine which sides to render [S1, unofficial]. To further optimize, meshes of all cubes were combined into a single big mesh called a 'chunk' [S1, unofficial]. This introduced restrictions such as Unity's limit of ~65,000 vertices per mesh, requiring chunks to be stacked or split [S1, unofficial].

## Sources
[S1] Voxel-Based Procedural Terrain Generation | PDF | Voxel | Rendering ..., https://www.scribd.com/document/840779393/Procedural-Terrain-Generation-Using-Voxels, fetched 2026-10-29 (unofficial)
[S2] gabkhanfig/Cube, https://github.com/gabkhanfig/Cube, fetched 2026-10-27 (unofficial)
[S3] Supporting Transparent Voxel Textures, https://github.com/TanTanDev/binary_greedy_mesher_demo/issues/5, fetched 2026-10-27 (unofficial)
[S4] ldominiq/ft_minecraft, https://github.com/ldominiq/ft_minecraft, fetched 2026-10-27 (unofficial)
