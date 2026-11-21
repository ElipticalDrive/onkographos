# Face Culling and Visibility Optimization

Also called: Interior Polygon Removal, Removing Interior Polygons

## Overview
Face Culling and Visibility Optimization reduces rendering load in blocky voxel meshes by reconstructing cubes from triangles and removing sides that are adjacent to solid neighbors, thereby avoiding the rendering of hidden geometry [S1, unofficial]. It solves the problem of naively rendering all 6 faces of every cube, including those hidden by neighbors, which wastes GPU cycles on up to 12 triangles per cube when only 2 (one side) are visible [S1, unofficial]. It sets itself apart from its parent [Blocky meshing](31-blocky-meshing.md) by focusing specifically on the elimination of internal faces during mesh assembly, and from siblings like [Greedy face merging](8935-greedy-face-merging.md) by operating on individual face visibility rather than merging adjacent quads into larger polygons.

## Shared principles
The core mechanism involves determining the visibility of each of the six faces of a voxel block based on the occupancy state of its immediate neighbors in the 3D grid. A face is considered visible if the adjacent voxel in the direction of the face's normal is empty (air) or transparent; it is culled if the neighbor is solid. This check is performed during mesh generation or update, ensuring that only exposed geometry is added to the vertex buffer. The optimization relies on the assumption that voxels are axis-aligned cubes and that visibility is determined by local topology.

## Subtopics

### [Voxel Block Placement with Face Culling](923-voxel-block-placement-with-face-culling.md)
A dynamic meshing technique that maintains the visibility of voxel chunk faces by toggling shared geometry state during block insertion or removal, rather than regenerating the entire mesh. It differs from its parent by integrating the face visibility state directly into the block data structure's bit fields, allowing for $O(1)$ local updates instead of global reconstruction. It differs from siblings like [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md) by relying on explicit neighbor occupancy checks within a fixed-size chunk rather than pre-computed coverage masks.

### [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md)
A method for removing hidden faces of partial voxel shapes (such as stairs, slabs, or custom geometries) by representing face coverage as a bit mask. It solves the problem where a boolean "solid/empty" flag is insufficient because partial shapes have mixed visible and hidden areas on a single face. Unlike sibling methods that rely on coarse block visibility or vertex packing, this method operates at the sub-face granularity, allowing precise culling of geometry that is occluded by neighbors while preserving exposed portions of the same face.

### [ChunkMesh Vertex Packing and Management](3566-chunkmesh-vertex-packing-and-management.md)
A dynamic mesh generation and update strategy for voxel terrain that combines face culling with compact vertex storage and incremental buffer updates. It solves the problem of high memory bandwidth and draw call overhead in voxel rendering by packing vertex attributes into two 32-bit integers and managing mesh topology changes (addition/removal of blocks) through local face interaction logic. It sets itself apart from static meshing methods by supporting real-time incremental updates via a free-list vertex allocator, and from pure geometry shaders by performing all topology calculations on the CPU.

### [Voxel Backface Culling and Internal Face Masking](4569-voxel-backface-culling-and-internal-face-masking.md)
This method optimizes voxel mesh rendering by rejecting faces that are either occluded by adjacent voxels (internal faces) or oriented away from the camera (backfaces). It distinguishes itself from standard neighbor-check culling by performing visibility tests in projection space to determine front/back orientation, while using a compact bitmask to track internal occlusion, allowing for efficient geometry shader-based culling.

### [Block Visibility Culling](5334-block-visibility-culling.md)
Block Visibility Culling determines which voxel blocks require mesh updates or rendering based on the camera's position and rotation, avoiding unnecessary geometry generation when the camera has not moved significantly or changed its primary viewing direction. Unlike sibling methods that operate at the fine-grained face level within a block, this method operates at the coarse block level, using camera-relative coordinates to cull entire blocks or specific faces based on surface orientation relative to the player. It builds on [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md) by adding camera-relative visibility checks.

### [Extraction of Surface Points (Sobierajski)](5721-extraction-of-surface-points-sobierajski.md)
A fast display method for direct rendering of boundary surfaces by extracting only visible boundary voxels from volume data. It differs from sibling methods like [Face Culling](8735-face-culling.md) which operate on mesh primitives after assembly; Sobierajski's method operates on the voxel grid directly, projecting voxels to the image plane to avoid holes and using viewing-direction-based culling to reduce the list size before rendering.

### [Mask Projection Culling](6836-mask-projection-culling.md)
Mask Projection Culling is a visibility-based mesh filtering technique that removes vertices and faces not consistently observed across multiple camera poses by projecting 3D points into camera space and comparing their depth against rendered depth maps or binary masks. It differs from neighbor-based face culling by relying on external camera viewpoints rather than local voxel topology, and from simple back-face culling by using explicit depth/mask validation rather than geometric normals alone.

### [Mesh Culling via 3D Projection and Connected Components](6963-mesh-culling-via-3d-projection-and-connected-components.md)
This method culls mesh geometry by comparing projected vertex depths against reference depth maps rendered from multiple camera poses, classifying points as seen, unseen, or forecast to remove artifacts from unbounded Gaussian splatting reconstruction. It sets itself apart from sibling methods like [Face Culling](8735-face-culling.md) which rely on voxel neighbor occupancy, by using explicit multi-view visibility constraints derived from camera projections rather than local topological checks.

### [Face Culling](8735-face-culling.md)
Face Culling is a mesh generation optimization for blocky voxel meshes that reduces the number of vertices sent to the GPU by skipping the rendering of voxel faces that are adjacent to existing neighboring voxels. It solves the problem of naively rendering all 6 faces of every cube, including those hidden by neighbors. It differs from its siblings by performing a direct per-voxel neighbor check during mesh assembly to determine face visibility, rather than using coverage masks, backface culling, or opacity-based thresholds.

### [Opacity-based Mesh Culling](8759-opacity-based-mesh-culling.md)
Opacity-based mesh culling reduces the complexity of polygonal mesh representations of volumetric scenes by discarding geometric primitives that contribute negligible visual information. It solves the problem of excessive mesh count and storage overhead in neural radiance field (NeRF) or similar implicit-to-explicit conversions. Unlike sibling methods that rely on voxel occupancy or spatial connectivity, this method evaluates the optical depth (opacity) of individual triangles or faces against a threshold, retaining only those that significantly affect the final image.

### [From-Region Visibility Preprocessing](9048-from-region-visibility-preprocessing.md)
From-region visibility preprocessing is a technique that precomputes which parts of a 3D scene are visible from specific viewing regions to accelerate rendering. It solves the problem of expensive per-frame visibility determination for large datasets by shifting the computational load to an offline phase. Unlike sibling methods that rely on axis-aligned voxel occupancy checks or simple backface culling, this approach handles complex geometry and occluder fusion, though it incurs significant preprocessing overhead.

### Buried Bitmask Optimization
A bitmask used to track which blocks in a chunk are 'buried' (fully surrounded by opaque blocks), allowing the renderer to skip mesh generation for those blocks [S2, unofficial]. The algorithm resets the buried bitmask for the chunk, creates `MappedAdjacentChunks` for the current chunk position to access neighboring chunks, and iterates through all blocks in the chunk (0 to `CHUNK_SIZE`). For each block, it determines if it is buried by calling `block->isBuried(...)` and sets the corresponding flag in the bitmask. If all blocks in the chunk are buried (checked via `buriedBitmask->AreAllBlocksSet()`), the mesh is marked as empty and mesh generation is skipped. When generating the mesh, any block that is air or has its buried flag set in the bitmask is skipped [S2, unofficial].

### Two-pass transparent meshing with opaque separation
A meshing strategy that separates opaque and transparent geometry into two passes to ensure correct rendering order and reduce overdraw, by generating opaque faces between air/transparent blocks and transparent faces only for blocks touching air [S3, unofficial]. The algorithm first checks if the chunk contains any transparent blocks; if not, the second pass is skipped. In the first (opaque) pass, if the chunk contains transparent blocks, it tests for opaque blocks (ignoring transparent and air) to generate opaque faces between air and transparent blocks; otherwise, it tests for non-air blocks. In the second (transparent) pass, it tests for non-air blocks to mark only transparent blocks, generating faces for all transparent blocks touching air but not adding faces between transparent and solid blocks. Solid block types are filtered out during the transparent pass by ignoring any block types that are not transparent before constructing the AO value and adding the block type to the hashmap [S3, unofficial].

### Smart Leaf Rendering
A leaf block rendering strategy that preserves alpha cutouts on outer leaf surfaces while culling internal faces to reduce vertex count [S4, unofficial]. The behavior is controlled by a global static enum `sLeafRenderMode`, which defaults to `ChunkRenderer::LeafRenderMode::Smart` [S4, unofficial].

## Background
The concept of removing interior polygons to improve rendering performance originated in early voxel-based procedural terrain generation systems, such as those used in Minecraft-like games. The initial brute-force approach of instantiating models for every cube resulted in unnecessary rendering of hidden sides. To optimize this, a class was introduced to reconstruct cubes from triangles and check for solid neighbors, preventing the rendering of sides adjacent to them. This led to the combination of individual cube meshes into larger "chunks" to reduce draw calls, although this introduced vertex count limits (e.g., ~65,000 vertices per mesh in Unity) [S1, unofficial]. The technique has since evolved into various specialized forms, including dynamic chunk updates, coverage masks for partial shapes, and camera-relative culling, to address specific performance bottlenecks in modern voxel engines.

## Sources
[S1] Voxel-Based Procedural Terrain Generation | PDF | Voxel | Rendering ..., https://www.scribd.com/document/840779393/Procedural-Terrain-Generation-Using-Voxels, fetched 2026-10-29 (unofficial)
[S2] gabkhanfig/Cube, https://github.com/gabkhanfig/Cube, fetched 2026-10-27 (unofficial)
[S3] Supporting Transparent Voxel Textures, https://github.com/TanTanDev/binary_greedy_mesher_demo/issues/5, fetched 2026-10-27 (unofficial)
[S4] ldominiq/ft_minecraft, https://github.com/ldominiq/ft_minecraft, fetched 2026-10-27 (unofficial)
