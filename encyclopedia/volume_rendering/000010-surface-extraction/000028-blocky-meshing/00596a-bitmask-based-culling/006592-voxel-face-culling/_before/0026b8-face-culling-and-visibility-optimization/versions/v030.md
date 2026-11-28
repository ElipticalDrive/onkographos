# Face Culling and Visibility Optimization

Also called: Interior Polygon Removal, Removing Interior Polygons

## Overview
Face Culling and Visibility Optimization reduces rendering load by reconstructing voxel geometry from triangles and removing faces that are adjacent to solid neighbors, thereby avoiding the rendering of hidden interior polygons [S1, unofficial]. It solves the problem of naive mesh generation where every voxel contributes 6 faces (12 triangles) regardless of occlusion, which is inefficient when most faces are internal [S1, unofficial]. It sets itself apart from its siblings by operating primarily on local neighbor occupancy checks during mesh creation to eliminate internal faces, whereas methods like [Mask Projection Culling](6836-mask-projection-culling.md) or [Mesh Culling via 3D Projection and Connected Components](6963-mesh-culling-via-3d-projection-and-connected-components.md) rely on global camera visibility, and [Opacity-based Mesh Culling](8759-opacity-based-mesh-culling.md) relies on material transparency thresholds.

## Shared principles
The core mechanism involves iterating over occupied voxel positions and evaluating each of the six potential face directions. A face is generated only if the adjacent voxel in that direction is empty, transparent (depending on configuration), or non-existent. If the adjacent voxel is solid and opaque, the face is considered internal and is culled. This reduces the geometry count from $6 \times N$ faces to approximately $2 \times N$ faces for large solid structures, as internal faces are never generated [S1, unofficial].

## Subtopics

### [Voxel Block Shape & Culling Configuration](2176-voxel-block-shape-culling-configuration.md)
Defines per-block geometry and transparency culling modes (e.g., merged vs. unmerged transparent groups).

### [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md)
Uses bitmasks to cull partially occluded faces from partial shapes like stairs and slabs.

### [ChunkMesh Vertex Packing and Management](3566-chunkmesh-vertex-packing-and-management.md)
Manages compact vertex storage and GPU buffer updates for dynamic mesh changes.

### [Voxel Backface Culling and Internal Face Masking](4569-voxel-backface-culling-and-internal-face-masking.md)
Adds view-dependent backface culling and bit-packed internal face masks.

### [Block Visibility Culling](5334-block-visibility-culling.md)
Skips mesh updates for entire blocks if the camera position and viewing direction have not changed significantly.

### [Extraction of Surface Points (Sobierajski)](5721-extraction-of-surface-points-sobierajski.md)
Extracts visible boundary voxels directly from volume data for direct rendering, bypassing mesh generation.

### [Mask Projection Culling](6836-mask-projection-culling.md)
Filters mesh vertices based on visibility across multiple camera poses using depth maps.

### [Mesh Culling via 3D Projection and Connected Components](6963-mesh-culling-via-3d-projection-and-connected-components.md)
Removes geometric artifacts and occluded regions by projecting mesh vertices into multiple camera views.

### [Face Culling](8735-face-culling.md)
The base method of skipping voxel faces adjacent to existing neighboring voxels.

### [Opacity-based Mesh Culling](8759-opacity-based-mesh-culling.md)
Discards faces with negligible opacity values from polygonal meshes.

### [From-Region Visibility Preprocessing](9048-from-region-visibility-preprocessing.md)
Precomputes visibility for large models to discard occluded geometry at runtime.

### Buried Bitmask Optimization
A bitmask used to track which blocks in a chunk are 'buried' (fully surrounded by opaque blocks), allowing the renderer to skip mesh generation for those blocks [S2, unofficial]. The process resets the buried bitmask for the chunk, creates `MappedAdjacentChunks` for the current chunk position to access neighboring chunks, and iterates through all blocks in the chunk (0 to `CHUNK_SIZE`) to determine if each block is buried by calling `block->isBuried(...)` and setting the corresponding flag in the bitmask [S2, unofficial]. If all blocks in the chunk are buried (checked via `buriedBitmask->AreAllBlocksSet()`), the mesh is marked as empty and mesh generation is skipped [S2, unofficial]. When generating the mesh, any block that is air or has its buried flag set in the bitmask is skipped [S2, unofficial].

### Two-pass transparent meshing with opaque separation
A meshing strategy that separates opaque and transparent geometry into two passes to ensure correct rendering order and reduce overdraw, by generating opaque faces between air/transparent blocks and transparent faces only for blocks touching air [S3, unofficial]. The method first checks if the chunk contains any transparent blocks; if not, the second pass is skipped entirely [S3, unofficial]. In the first pass (opaque), if the chunk contains transparent blocks, the system tests for opaque blocks (ignoring transparent and air) to generate opaque faces between air and transparent blocks; otherwise, it tests for non-air blocks [S3, unofficial]. In the second pass (transparent), the system tests for non-air blocks to mark only transparent blocks, generating faces for all transparent blocks touching air but not adding faces between transparent and solid blocks (since solid blocks already have those faces) [S3, unofficial]. To filter out solid block types during the transparent pass, the system checks if it is in the transparent pass before constructing the AO value and adding the block type to the hashmap; if so, it ignores any block types that are not transparent [S3, unofficial].

### Smart Leaf Rendering
A leaf block rendering strategy that preserves alpha cutouts on outer leaf surfaces while culling internal faces to reduce vertex count [S4, unofficial]. The behavior is controlled by a global static enum `sLeafRenderMode`, which defaults to `ChunkRenderer::LeafRenderMode::Smart` [S4, unofficial].

## Background
The concept of removing interior polygons was introduced in the context of voxel-based procedural terrain generation, specifically referencing Minecraft-like chunk systems [S1, unofficial]. Early implementations used brute-force instantiation of cube models, which rendered all 6 sides (12 triangles) per cube regardless of visibility [S1, unofficial]. To optimize this, a class was added to reconstruct cubes from triangles and check for solid neighbors; if a neighbor existed, the adjacent side was not rendered [S1, unofficial]. This reduced the rendering load significantly, as only the visible sides (e.g., 2 triangles for 1 side) were drawn [S1, unofficial]. The meshes of all cubes were then combined into a single big mesh called a 'chunk' to further optimize implementation, though this introduced engine-specific vertex limits (e.g., ~65,000 vertices in Unity) [S1, unofficial].

## Sources
[S1] Voxel-Based Procedural Terrain Generation | PDF | Voxel | Rendering ..., https://www.scribd.com/document/840779393/Procedural-Terrain-Generation-Using-Voxels, fetched 2026-10-29 (unofficial)
[S2] gabkhanfig/Cube, https://github.com/gabkhanfig/Cube, fetched 2026-10-27 (unofficial)
[S3] Supporting Transparent Voxel Textures, https://github.com/TanTanDev/binary_greedy_mesher_demo/issues/5, fetched 2026-10-27 (unofficial)
[S4] ldominiq/ft_minecraft, https://github.com/ldominiq/ft_minecraft, fetched 2026-10-27 (unofficial)
