# Face Culling and Visibility Optimization

Also called: Interior Polygon Removal, Removing Interior Polygons

## Overview
Face Culling and Visibility Optimization reduces the rendering load in blocky meshing by reconstructing voxel geometry from triangles and removing faces that are adjacent to solid neighbors, thereby avoiding the rendering of hidden geometry [S1, unofficial]. It solves the problem of naively rendering all 6 faces (12 triangles) of every voxel, including those occluded by adjacent blocks, by only generating faces for "exposed" boundaries [S1, unofficial]. Unlike sibling methods that may use complex projection masks, backface culling, or opacity thresholds, this family primarily relies on direct spatial neighbor checks during mesh creation to identify visible surfaces [S1, unofficial].

## Shared principles
The core mechanism common to most methods in this family is the evaluation of voxel adjacency to determine face visibility.

1.  **Neighbor Lookup:** For each voxel at position $P$ and each of the 6 face directions $d \in \{\pm X, \pm Y, \pm Z\}$, the algorithm checks the occupancy of the neighbor at $P + d$.
2.  **Visibility Condition:** A face is generated if and only if the neighbor is empty (air) or transparent (depending on the specific culling mode). If the neighbor is solid/opaque, the face is internal and culled.
3.  **Mesh Assembly:** Visible faces are collected into a chunk mesh. To optimize GPU uploads, vertices and indices are often packed into dense formats (e.g., 32-bit integers) and managed via free-list allocators to support incremental updates [S1, unofficial].

## Subtopics

### [Voxel Block Shape & Culling Configuration](2176-voxel-block-shape-culling-configuration.md)
A trait-based system that defines block geometry, transparency culling modes, and lighting properties per block type. It solves the problem of heterogeneous block types (opaque, transparent, fluid, complex shapes) requiring different mesh generation behaviors by centralizing rules in a single interface. Unlike generic meshing, it allows per-block customization of shape (e.g., ramps) and culling strategy (e.g., merging transparent faces).

### [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md)
A method for removing hidden faces of partial voxel shapes (stairs, slabs) by representing face coverage as a 16-bit bitmask. It solves the problem where a boolean "solid/empty" flag is insufficient for partial shapes. Unlike sibling methods that rely on coarse block visibility, this operates at sub-face granularity, allowing precise culling of occluded portions while preserving exposed areas.

### [ChunkMesh Vertex Packing and Management](3566-chunkmesh-vertex-packing-and-management.md)
A dynamic mesh generation and memory management technique that packs vertex attributes into dense 32-bit integer formats and manages buffer updates via a free-list allocator. It solves the problem of high memory overhead and frequent GPU buffer uploads by minimizing vertex data size and enabling efficient incremental updates. It differs from static meshing by supporting real-time block modification.

### [Voxel Backface Culling and Internal Face Masking](4569-voxel-backface-culling-and-internal-face-masking.md)
This method optimizes rendering by rejecting faces that are either occluded by adjacent voxels (internal faces) or oriented away from the camera (backfaces). It distinguishes itself by performing visibility tests in projection space to determine front/back orientation, while using a compact bitmask to track internal occlusion, allowing for efficient geometry shader-based culling.

### [Block Visibility Culling](5334-block-visibility-culling.md)
Determines which voxel blocks require mesh updates based on the camera's position and rotation, avoiding unnecessary geometry generation when the camera has not moved significantly. It operates at the coarse block level, using camera-relative coordinates and surface orientation to cull invisible faces before they are submitted to the graphics pipeline. Unlike sibling methods that operate at the fine-grained face level, this relies on simple integer block coordinate comparison and basic trigonometric orientation checks.

### [Extraction of Surface Points (Sobierajski)](5721-extraction-of-surface-points-sobierajski.md)
A fast display method for direct rendering of boundary surfaces by extracting only visible boundary voxels from volume data. It sets itself apart by relying on a 2D boundary tracing algorithm on axis-aligned slices to determine visibility from a fixed set of viewing directions (typically six orthographic views), rather than using complex 3D connectivity or ray-casting.

### [Mask Projection Culling](6836-mask-projection-culling.md)
A visibility-based mesh filtering technique that removes faces and vertices not consistently observed across multiple camera poses by projecting points into camera space and comparing their depth against rendered depth maps or binary occupancy masks. It solves the problem of retaining only geometry visible from specific viewpoints, effectively discarding occluded or background noise. It differs from neighbor-based culling by relying on explicit camera projections and depth/mask sampling.

### [Mesh Culling via 3D Projection and Connected Components](6963-mesh-culling-via-3d-projection-and-connected-components.md)
This method culls mesh geometry by projecting vertices into multiple camera views and comparing their depths against reference depth maps to identify and remove unseen or artifact geometry. It distinguishes itself from neighbor-based culling by relying on explicit visibility testing against a set of camera poses rather than local voxel occupancy, making it suitable for unbounded or sparse reconstructions where topological connectivity is unreliable.

### [Face Culling](8735-face-culling.md)
Face Culling is a mesh generation optimization that reduces the number of vertices sent to the GPU by skipping the rendering of voxel faces that are adjacent to existing neighboring voxels. It solves the problem of naively rendering all 6 faces of every cube, including those hidden by neighbors. It is distinguished from its siblings by its reliance on direct spatial neighbor checks during the mesh creation phase to identify "exposed" faces, rather than using coverage masks, backface normals, or block-level visibility pre-processing.

### [Opacity-based Mesh Culling](8759-opacity-based-mesh-culling.md)
Opacity-based Mesh Culling is a post-processing optimization technique that reduces the geometric complexity of a polygon mesh by discarding faces that contribute negligible visual information based on their opacity values. It solves the problem of excessive mesh density in neural radiance field (NeRF) or implicit surface extractions, where many triangles represent transparent or low-density regions. Unlike spatial culling methods that rely on neighbor occupancy, this method operates on the material property (opacity) of the mesh faces themselves, retaining only those that meet a specific opacity threshold.

### [From-Region Visibility Preprocessing](9048-from-region-visibility-preprocessing.md)
From-region visibility preprocessing is a technique that precomputes the set of visible geometry for specific viewing regions or viewpoints to accelerate rendering. It solves the problem of expensive per-frame visibility testing for large datasets by shifting the computational burden to an offline phase. Unlike [Face Culling](8735-face-culling.md) which operates on local voxel adjacency, this method considers global occlusion relationships, often involving complex occluder fusion, but historically suffers from significant preprocessing overhead.

### Buried Bitmask Optimization
A bitmask used to track which blocks in a chunk are 'buried' (fully surrounded by opaque blocks), allowing the renderer to skip mesh generation for those blocks [S2, unofficial]. The algorithm resets the buried bitmask for the chunk, creates mapped adjacent chunks for neighbor access, and iterates through all blocks to determine if they are buried. If all blocks in a chunk are buried, the mesh is marked as empty and generation is skipped. When generating the mesh, any block that is air or has its buried flag set is skipped [S2, unofficial].

### Two-pass transparent meshing with opaque separation
A meshing strategy that separates opaque and transparent geometry into two passes to ensure correct rendering order and reduce overdraw, by generating opaque faces between air/transparent blocks and transparent faces only for blocks touching air [S3, unofficial]. The first pass (opaque) generates faces for opaque blocks, testing for opaque blocks if transparent blocks exist in the chunk (ignoring transparent and air), otherwise testing for non-air. This generates opaque faces between air and transparent blocks. The second pass (transparent) tests for non-air blocks to mark only transparent blocks, generating faces for all transparent blocks touching air but not adding faces between transparent and solid blocks. Solid block types are filtered out during the transparent pass by ignoring non-transparent block types before constructing attributes [S3, unofficial].

### Smart Leaf Rendering
A leaf block rendering strategy that preserves alpha cutouts on outer leaf surfaces while culling internal faces to reduce vertex count [S4, unofficial]. It is controlled by a global static enum `sLeafRenderMode`, which defaults to `ChunkRenderer::LeafRenderMode::Smart` [S4, unofficial].

## Background
The concept of removing interior polygons to improve rendering performance originated in early voxel-based terrain generation, such as in Minecraft-like engines [S1, unofficial]. The initial brute-force approach of instantiating cube models was replaced by a class that reconstructs cubes from triangles, allowing each cube to check for solid neighbors and determine which sides to render [S1, unofficial]. To further optimize, meshes of all cubes were combined into a single big mesh called a 'chunk' [S1, unofficial]. This introduced constraints, such as Unity's limit of ~65,000 vertices per mesh, leading to techniques like stacking chunks [S1, unofficial]. The method has since evolved to include more sophisticated culling strategies for transparency, partial shapes, and GPU-based optimization [S1, S2, S3, S4, unofficial].

## Sources
[S1] Voxel-Based Procedural Terrain Generation | PDF | Voxel | Rendering ..., https://www.scribd.com/document/840779393/Procedural-Terrain-Generation-Using-Voxels, fetched 2026-10-29 (unofficial)
[S2] gabkhanfig/Cube, https://github.com/gabkhanfig/Cube, fetched 2026-10-27 (unofficial)
[S3] Supporting Transparent Voxel Textures, https://github.com/TanTanDev/binary_greedy_mesher_demo/issues/5, fetched 2026-10-27 (unofficial)
[S4] ldominiq/ft_minecraft, https://github.com/ldominiq/ft_minecraft, fetched 2026-10-27 (unofficial)
