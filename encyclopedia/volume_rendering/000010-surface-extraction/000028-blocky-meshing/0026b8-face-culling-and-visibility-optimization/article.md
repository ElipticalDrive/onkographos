# Face Culling and Visibility Optimization

Also called: Interior Polygon Removal, Removing Interior Polygons

## Overview
Face Culling and Visibility Optimization is a family of techniques that reduce the rendering load of blocky voxel meshes by reconstructing cubes from triangles and removing sides that are adjacent to solid neighbors, thereby avoiding the rendering of 12 triangles (6 sides) when only 2 triangles (1 side) are visible [S1, unofficial]. It solves the problem of naively rendering all faces of every voxel, including those hidden by neighbors, which wastes GPU bandwidth and fill-rate. What sets this family apart from its siblings is its primary focus on **local spatial adjacency** (neighbor checks) to determine face visibility during mesh generation or update, as opposed to post-generation projection culling or camera-relative block visibility. It builds on [Blocky meshing](31-blocky-meshing.md) by optimizing the output of the meshing process.

## Shared principles
The core mechanism common to all methods in this family is the determination of face visibility based on the occupancy state of neighboring voxels.

1.  **Neighbor Check**: For every voxel face (typically 6 per voxel), the algorithm queries the state of the adjacent voxel in the direction of the face's normal vector $\vec{n}$.
2.  **Visibility Logic**:
    *   If the neighbor is **empty** (air/void), the face is **visible** and must be rendered.
    *   If the neighbor is **solid** (opaque block), the face is **hidden** (internal) and must be culled.
    *   If the neighbor is **transparent/translucent**, the visibility depends on the specific material handling (e.g., [Two-pass transparent meshing with opaque separation](#two-pass-transparent-meshing-with-opaque-separation) or [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md)).
3.  **Mesh Reconstruction**: Instead of instantiating individual cube models, the algorithm reconstructs the visible geometry (triangles) for the exposed faces and combines them into a single mesh buffer (chunk) to minimize draw calls [S1, unofficial].

## Subtopics

### [Voxel Block Placement with Face Culling](923-voxel-block-placement-with-face-culling.md)
A technique for dynamically updating a voxel chunk's internal state and face visibility when blocks are placed or removed, using neighbor checks to toggle shared faces without regenerating the entire mesh. It differs from [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md) by operating on discrete block indices and explicit face toggles rather than coverage masks, and from [ChunkMesh Vertex Packing and Management](3566-chunkmesh-vertex-packing-and-management.md) by focusing on the logical state management of block existence and face visibility rather than vertex buffer layout optimization. Choose this when implementing dynamic voxel editing (Minecraft-style) where incremental updates are required.

### [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md)
A method for removing hidden faces of partial voxel shapes (such as stairs, slabs, or custom geometries) by representing face coverage as a bit mask. It solves the problem where a boolean "solid/empty" flag is insufficient because partial shapes have mixed visible and hidden areas on a single face. Unlike sibling methods that rely on coarse block visibility or vertex packing, this method operates at the sub-face granularity, allowing precise culling of geometry that is occluded by neighbors while preserving exposed portions of the same face. Choose this for non-cubic voxel shapes.

### [ChunkMesh Vertex Packing and Management](3566-chunkmesh-vertex-packing-and-management.md)
A dynamic mesh generation and update technique for blocky voxel worlds that stores vertex data in highly compacted 32-bit integer formats and manages incremental mesh changes via a reusable vertex slot pool. It solves the problem of high memory bandwidth and buffer update overhead in real-time voxel rendering by minimizing the number of vertices and indices transferred to the GPU per frame. It differs from static meshing methods by supporting arbitrary block addition and removal with localized face culling, and from simple face culling by explicitly managing vertex reuse and buffer synchronization. Choose this for performance-critical real-time engines with frequent chunk updates.

### [Voxel Backface Culling and Internal Face Masking](4569-voxel-backface-culling-and-internal-face-masking.md)
A visibility optimization technique for blocky voxel meshing that rejects invisible faces by determining front/back orientation in projection space and masking faces shared between neighboring voxels. It builds on [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md) by adding a projection-space backface test and a bitmask-based internal face rejection, allowing efficient culling of both back-facing and internal surfaces in a single pass. Choose this when rendering large chunks where backface culling provides significant performance gains.

### [Block Visibility Culling](5334-block-visibility-culling.md)
A method to determine which blocks need to be rendered based on camera position and rotation, avoiding unnecessary mesh updates when the camera has not moved significantly or changed its primary viewing direction. It renders individual blocks by calculating screen coordinates relative to the player, determining surface orientation (top/bottom/front/back), and culling invisible faces based on player position and block visibility flags. It sets itself apart from sibling methods by focusing on **camera-relative block-level culling** and **orientation-based face skipping** rather than just neighbor-based face culling (as in [Voxel Block Placement with Face Culling](923-voxel-block-placement-with-face-culling.md)) or global mesh projection (as in [Mesh Culling via 3D Projection and Connected Components](6963-mesh-culling-via-3d-projection-and-connected-components.md)). Choose this for simple block-based rendering where camera-relative optimization is sufficient.

### [Extraction of Surface Points (Sobierajski)](5721-extraction-of-surface-points-sobierajski.md)
A fast display method for direct rendering of boundary surfaces by extracting only visible boundary voxels from volume data. It differs from standard marching cubes or raycasting by operating in object space: it pre-computes a list of visible surface points (voxels) based on viewing directions, then projects them to the image plane. This avoids per-pixel ray traversal but requires handling projection artifacts like holes. Choose this for direct volume visualization where point-based rendering is acceptable.

### [Mask Projection Culling](6836-mask-projection-culling.md)
A visibility optimization technique that filters mesh vertices and faces by projecting them into multiple camera views and checking if they fall within dilated binary masks, removing geometry not visible in any view. It differs from neighbor-based face culling (e.g., [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md)) by relying on camera-space projection rather than spatial adjacency, making it suitable for complex scenes where occlusion is view-dependent rather than purely topological. Choose this for multi-view reconstruction or complex occlusion scenarios.

### [Mesh Culling via 3D Projection and Connected Components](6963-mesh-culling-via-3d-projection-and-connected-components.md)
This method culls mesh geometry by projecting vertices into multiple camera views and comparing their depths against reference depth maps to identify and remove unseen or forecast geometry. It distinguishes itself from neighbor-based face culling [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md) by relying on global visibility from arbitrary viewpoints rather than local voxel adjacency, making it suitable for removing artifacts from unbounded reconstructions. Choose this for post-processing reconstructed meshes from multi-view stereo or NeRF.

### [Face Culling](8735-face-culling.md)
Face Culling is a mesh generation optimization that reduces the number of vertices and triangles sent to the GPU by skipping the rendering of voxel faces that are adjacent to existing neighboring voxels. It solves the problem of naively rendering all 6 faces of every cube, including those hidden by neighbors. It sets itself apart from its siblings by operating on a per-voxel neighbor check basis during mesh creation, rather than using coverage masks, block visibility, or post-generation projection. Choose this as the standard baseline for voxel meshing.

### Buried Bitmask Optimization
A bitmask used to track which blocks in a chunk are 'buried' (fully surrounded by opaque blocks), allowing the renderer to skip mesh generation for those blocks. The process involves resetting the buried bitmask for the chunk, creating `MappedAdjacentChunks` for the current chunk position to access neighboring chunks, and iterating through all blocks in the chunk to determine if each is buried by calling `block->isBuried(...)`. If all blocks in the chunk are buried (checked via `buriedBitmask->AreAllBlocksSet()`), the mesh is marked as empty and generation is skipped. When generating the mesh, any block that is air or has its buried flag set in the bitmask is skipped. This optimization is particularly effective in dense, enclosed structures [S2, unofficial].

### Two-pass transparent meshing with opaque separation
A meshing strategy that separates opaque and transparent geometry into two passes to ensure correct rendering order and reduce overdraw, by generating opaque faces between air/transparent blocks and transparent faces only for blocks touching air. The first pass (opaque) generates faces for opaque blocks, testing for opaque blocks if transparent blocks exist in the chunk (ignoring transparent and air), otherwise testing for non-air blocks. This generates opaque faces between air and transparent blocks. The second pass (transparent) tests for non-air blocks to mark only transparent blocks, generating faces for all transparent blocks touching air, but not adding faces between transparent and solid blocks (since solid blocks already have those faces). Solid block types are filtered out during the transparent pass to avoid incorrect transparency rendering [S3, unofficial].

### Opacity-based Mesh Culling
A technique that clips triangular faces if the opacity of all intersection points on that face is below 0.3, retaining only meshes meeting this criterion. This reduces the mesh count to 5% of the original. It also involves coordinate transformation to align the obtained mesh with real-world scale and mapping vertex coordinates of triangles to corresponding positions in UV coordinates. This method is used in conjunction with neural radiance fields for real-time rendering of large-scale scenes [S4].

### From-Region Visibility Preprocessing
A from-region visibility preprocessing algorithm that could handle 3D environments. However, it required long preprocessing times (33 hours for a model with 6 million triangles) [S5, unofficial].

## Background
The concept of removing interior polygons to improve rendering performance was introduced in the context of voxel-based procedural terrain generation, specifically referencing Minecraft-like worlds [S1, unofficial]. The initial optimization involved reconstructing cubes from triangles and checking for solid neighbors to avoid rendering hidden sides [S1, unofficial]. This led to the combination of cube meshes into single "chunks" to optimize implementation, although this introduced engine-specific vertex limits (e.g., Unity's ~65,000 vertex limit per mesh) [S1, unofficial]. The technique has since evolved into various specialized forms, including bitmask optimizations for buried blocks [S2, unofficial], two-pass transparent meshing [S3, unofficial], and opacity-based culling for neural rendering [S4]. From-region visibility preprocessing, while applicable to 3D environments, has been noted for its high computational cost [S5, unofficial].

## Sources
[S1] Voxel-Based Procedural Terrain Generation | PDF | Voxel | Rendering ..., https://www.scribd.com/document/840779393/Procedural-Terrain-Generation-Using-Voxels, fetched 2026-10-29 (unofficial)
[S2] gabkhanfig/Cube, https://github.com/gabkhanfig/Cube, fetched 2026-10-27 (unofficial)
[S3] Supporting Transparent Voxel Textures, https://github.com/TanTanDev/binary_greedy_mesher_demo/issues/5, fetched 2026-10-27 (unofficial)
[S4] UE4-NeRF: Neural Radiance Field for Real-Time Rendering of Large-Scale Scene, https://doi.org/10.48550/arXiv.2310.13263, fetched 2026-10-29
[S5] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
