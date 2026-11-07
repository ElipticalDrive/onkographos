# Face Culling and Visibility Optimization

Also called: Interior Polygon Removal, Removing Interior Polygons

## Overview
Face Culling and Visibility Optimization reduces the rendering load in blocky voxel meshing by reconstructing cubes from triangles and removing sides that are adjacent to solid neighbors, thereby avoiding the rendering of hidden geometry [S1, unofficial]. It solves the problem of naively rendering all 6 faces (12 triangles) of every cube, including those hidden by neighbors, which results in excessive geometry and poor performance [S1, unofficial]. It sets itself apart from its parent [Blocky meshing](31-blocky-meshing.md) by focusing on the removal of interior polygons rather than just merging adjacent faces, and from [Boolean Voxel Meshing](971-boolean-voxel-meshing.md) by operating on discrete voxel adjacency rather than constructive solid geometry [G].

## Shared principles
The core mechanism common to all methods in this family is the determination of face visibility based on the occupancy state of neighboring voxels or the geometric relationship to the camera/viewer.

1.  **Neighbor Occupancy Check**: For a voxel at position $(x, y, z)$ with a face normal $\vec{n}$, the face is visible if and only if the neighbor at position $(x, y, z) + \vec{n}$ is empty (air/void) or transparent, depending on the material handling [S1, unofficial]. If the neighbor is solid, the face is internal and culled.
2.  **Mesh Reconstruction**: Instead of instantiating full cube models, the meshing process reconstructs individual faces (quads) only when they are determined to be visible [S1, unofficial].
3.  **Chunking**: To manage memory and rendering limits, visible faces are combined into larger meshes called 'chunks' [S1, unofficial].

## Subtopics

### [Voxel Block Placement with Face Culling](923-voxel-block-placement-with-face-culling.md)
A technique for dynamically updating a voxel chunk's internal state and face visibility when blocks are placed or removed, using neighbor checks to toggle shared faces without regenerating the entire mesh. It differs from [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md) by operating on discrete block indices and explicit face toggles rather than coverage masks [G], and from [ChunkMesh Vertex Packing and Management](3566-chunkmesh-vertex-packing-and-management.md) by focusing on the logical state management of block existence and face visibility rather than vertex buffer layout optimization [G].

### [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md)
A method for removing hidden faces of partial voxel shapes (such as stairs, slabs, or custom geometries) by representing face coverage as a bit mask. It solves the problem where a boolean "solid/empty" flag is insufficient because partial shapes have mixed visible and hidden areas on a single face. Unlike sibling methods that rely on coarse block visibility or vertex packing, this method operates at the sub-face granularity, allowing precise culling of geometry that is occluded by neighbors while preserving exposed portions of the same face.

### [ChunkMesh Vertex Packing and Management](3566-chunkmesh-vertex-packing-and-management.md)
A dynamic mesh generation and update technique for voxel terrain that minimizes memory bandwidth and buffer upload overhead by packing vertex attributes into dense 32-bit integer formats. It solves the problem of sparse, high-latency vertex buffer updates in incremental blocky meshing by using a reusable vertex index pool and bitwise-packed data structures, allowing efficient addition and removal of individual voxel faces without reallocating GPU memory. It differs from standard meshing by storing position, texture, and lighting data in two `uint32` values per vertex rather than separate floating-point arrays, and from static meshing by supporting runtime face culling and dynamic buffer updates via a `DYNAMIC_DRAW` hint.

### [Voxel Backface Culling and Internal Face Masking](4569-voxel-backface-culling-and-internal-face-masking.md)
A geometry optimization technique for blocky voxel meshing that rejects invisible faces by determining front/back orientation in projection space and masking faces shared between neighboring voxels. It sets itself apart from sibling methods like [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md) by performing visibility checks directly within the geometry shader using projection-space dot products and bit-packed voxel codes, rather than relying on pre-computed coverage masks or complex connected component analysis.

### [Block Visibility Culling](5334-block-visibility-culling.md)
Determines which voxel blocks require mesh updates or rendering based on the camera's position and rotation, avoiding unnecessary geometry processing when the camera has not moved significantly or changed its primary viewing direction. Unlike its parent [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md), which operates on individual voxel faces, this method operates at the coarse block level, using camera-relative orientation to cull entire block faces before they are processed by the renderer. It differs from [Voxel Face Visibility Check](6723-voxel-face-visibility-check.md) by evaluating visibility at the block coordinate level rather than the fine-grained face level within a block.

### [Extraction of Surface Points (Sobierajski)](5721-extraction-of-surface-points-sobierajski.md)
A fast display method for the direct rendering of boundary surfaces by extracting only visible boundary voxels from volume data. It solves the problem of rendering speed in direct volume rendering by avoiding the processing of hidden geometry. It sets itself apart from sibling methods like [Voxel Block Placement with Face Culling](923-voxel-block-placement-with-face-culling.md) by operating on a list of extracted surface points (voxels) rather than constructing a mesh of triangles, and by using viewing-direction-based visibility rather than static neighbor occupancy checks.

### [Mask Projection Culling](6836-mask-projection-culling.md)
A visibility optimization technique that filters mesh vertices and faces by projecting them into multiple camera views and checking if they fall within dilated binary masks, removing geometry not visible in any view. It sets itself apart from sibling methods like [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md) by operating in screen space using pre-computed or rendered masks rather than volumetric occupancy checks, and from [Voxel Backface Culling and Internal Face Masking](4569-voxel-backface-culling-and-internal-face-masking.md) by relying on multi-view consistency rather than local geometric orientation.

### [Mesh Culling via 3D Projection and Connected Components](6963-mesh-culling-via-3d-projection-and-connected-components.md)
This method culls mesh geometry by projecting 3D vertices into multiple 2D camera views and comparing their depths against reference depth maps to classify faces as seen, unseen, or forecast. It is specifically designed to remove artifacts from unbounded Gaussian splatting reconstruction where traditional voxel-based neighbor checks fail due to the lack of a bounded grid. Unlike sibling methods that rely on local voxel occupancy (e.g., [Face Culling](8735-face-culling.md)), this approach uses global visibility information from multiple viewpoints to determine face retention.

### [Face Culling](8735-face-culling.md)
A mesh generation optimization that reduces the number of vertices and triangles sent to the GPU by skipping the rendering of voxel faces that are adjacent to existing neighboring voxels. It solves the problem of naively rendering all 6 faces of every cube, including those hidden by neighbors, which results in excessive geometry and poor performance. It sets itself apart from siblings like [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md) by relying on direct neighbor occupancy checks rather than coverage masks, and from [Voxel Backface Culling and Internal Face Masking](4569-voxel-backface-culling-and-internal-face-masking.md) by focusing on inter-voxel adjacency rather than camera-relative backface removal.

### [From-Region Visibility Preprocessing](9048-from-region-visibility-preprocessing.md)
From-region visibility preprocessing is a technique that precomputes the visibility of geometric primitives (typically triangles) relative to specific viewing regions or points within a 3D environment. It solves the problem of efficiently culling occluded geometry during rendering by shifting the computational burden to an offline phase. Unlike methods that assume axis-aligned cells or require minimal preprocessing (such as [PLP](link) or [cPLP](link)), this approach handles complex, non-axis-aligned geometry but incurs significant preprocessing costs.

### Buried Bitmask Optimization
A bitmask used to track which blocks in a chunk are 'buried' (fully surrounded by opaque blocks), allowing the renderer to skip mesh generation for those blocks [S2, unofficial]. The algorithm resets the buried bitmask for the chunk, creates `MappedAdjacentChunks` for the current chunk position to access neighboring chunks, and iterates through all blocks in the chunk (0 to `CHUNK_SIZE`) to determine if each block is buried by calling `block->isBuried(...)` and setting the corresponding flag in the bitmask [S2, unofficial]. If all blocks in the chunk are buried (checked via `buriedBitmask->AreAllBlocksSet()`), the mesh is marked as empty and mesh generation is skipped [S2, unofficial]. When generating the mesh, any block that is air or has its buried flag set in the bitmask is skipped [S2, unofficial].

### Two-pass transparent meshing with opaque separation
A meshing strategy that separates opaque and transparent geometry into two passes to ensure correct rendering order and reduce overdraw, by generating opaque faces between air/transparent blocks and transparent faces only for blocks touching air [S3, unofficial]. The first pass (opaque) generates faces for opaque blocks; if the chunk contains transparent blocks, it tests for opaque blocks (ignoring transparent and air) to generate opaque faces between air and transparent blocks, otherwise it tests for non-air blocks [S3, unofficial]. The second pass (transparent) tests for non-air blocks to mark only transparent blocks, generating faces for all transparent blocks touching air but not adding faces between transparent and solid blocks (since solid blocks already have those faces) [S3, unofficial]. Solid block types are filtered out during the transparent pass by ignoring any block types that are not transparent before constructing the AO value and adding the block type to the hashmap [S3, unofficial].

### Opacity-based Mesh Culling
Clips triangular faces if the opacity of all intersection points on that face is below 0.3, retaining only meshes meeting this criterion, which reduces mesh count to 5% of the original [S4]. It also performs coordinate transformation to align the obtained mesh with real-world scale and maps vertex coordinates of triangles to corresponding positions in UV coordinates [S4].

## Background
The concept of removing interior polygons to improve rendering performance originated in early voxel-based procedural terrain generation systems, such as those used in Minecraft-like engines [S1, unofficial]. The initial brute-force approach of instantiating full cube models resulted in unnecessary rendering of hidden triangles; for example, rendering 12 triangles (6 sides) when only 2 triangles (1 side) were visible [S1, unofficial]. To address this, a class was added to reconstruct cubes from triangles and check for solid neighbors, preventing the rendering of sides adjacent to those neighbors [S1, unofficial]. This optimization was further enhanced by combining the meshes of all individual cubes into a single big mesh called a 'chunk' [S1, unofficial]. However, this introduced engine-specific restrictions, such as Unity's limit of ~65,000 vertices per mesh, requiring chunks to be stacked or split to achieve larger volumes [S1, unofficial].

## Sources

[S1] Voxel-Based Procedural Terrain Generation | PDF | Voxel | Rendering ..., https://www.scribd.com/document/840779393/Procedural-Terrain-Generation-Using-Voxels, fetched 2026-10-29 (unofficial)
[S2] gabkhanfig/Cube, https://github.com/gabkhanfig/Cube, fetched 2026-10-27 (unofficial)
[S3] Supporting Transparent Voxel Textures, https://github.com/TanTanDev/binary_greedy_mesher_demo/issues/5, fetched 2026-10-27 (unofficial)
[S4] UE4-NeRF: Neural Radiance Field for Real-Time Rendering of Large-Scale Scene, https://doi.org/10.48550/arXiv.2310.13263, fetched 2026-10-29
