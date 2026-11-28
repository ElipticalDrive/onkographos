# Face Culling and Visibility Optimization

Also called: Interior Polygon Removal, Removing Interior Polygons

## Overview
Face Culling and Visibility Optimization reduces the rendering load in voxel-based volume rendering by reconstructing cubes from triangles and removing sides that are adjacent to solid neighbors, thereby avoiding the rendering of 12 triangles (6 sides) when only 2 triangles (1 side) are visible [S1, unofficial]. It differs from its siblings by focusing on the geometric adjacency within the voxel grid to eliminate internal faces, whereas methods like [Block Visibility Culling](5334-block-visibility-culling.md) operate on entire blocks based on camera position, and [Opacity-based Mesh Culling](8759-opacity-based-mesh-culling.md) filters based on material properties.

## Shared principles
The core mechanism involves iterating through occupied voxel positions and examining their six adjacent neighbors. For each face of a voxel, the algorithm determines visibility based on the occupancy of the neighbor in the direction of the face's normal vector $\mathbf{n}$ [S1, unofficial].

1.  **Neighbor Check**: For a voxel at position $P_{current}$ with a face normal $\mathbf{n}$, calculate the neighbor position $P_{neighbor} = P_{current} + \mathbf{n}$ [S1, unofficial].
2.  **Visibility Determination**:
    *   If a solid voxel exists at $P_{neighbor}$, the face is internal and culled [S1, unofficial].
    *   If no solid voxel exists at $P_{neighbor}$, the face is exposed and rendered [S1, unofficial].
3.  **Mesh Assembly**: The meshes of all individual cubes are combined into a single big mesh called a 'chunk' to optimize implementation [S1, unofficial].

## Subtopics

### [Voxel Block Placement with Face Culling](923-voxel-block-placement-with-face-culling.md)
Voxel Block Placement with Face Culling is a dynamic meshing technique that maintains the visibility of voxel chunk faces by toggling shared geometry state during block insertion or removal, rather than regenerating the entire mesh. It differs from its parent [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md) by integrating the face visibility state directly into the block data structure's bit fields, allowing for $O(1)$ local updates instead of global reconstruction. It differs from siblings like [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md) by relying on explicit neighbor occupancy checks within a fixed-size chunk rather than pre-computed coverage masks.

### [Voxel Block Shape & Culling Configuration](2176-voxel-block-shape-culling-configuration.md)
Voxel Block Shape & Culling Configuration is a trait-based system that defines block geometry, transparency culling modes, and lighting properties per block type. It solves the problem of heterogeneous block rendering by allowing different geometry types (cubes, ramps, fluids) and rendering behaviors (opaque, transparent merged, transparent unmerged) to be configured via a single interface, rather than hard-coded logic. Unlike sibling methods that focus on mesh assembly or vertex packing, this method defines the *rules* for which faces are generated and how they are classified before meshing begins.

### [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md)
Face Coverage Mask Culling is a method for removing hidden faces of partial voxel shapes (such as stairs, slabs, or custom geometries) by representing face coverage as a bit mask. It solves the problem where a boolean "solid/empty" flag is insufficient because partial shapes have mixed visible and hidden areas on a single face. Unlike sibling methods that rely on coarse block visibility or vertex packing, this method operates at the sub-face granularity, allowing precise culling of geometry that is occluded by neighbors while preserving exposed portions of the same face.

### [ChunkMesh Vertex Packing and Management](3566-chunkmesh-vertex-packing-and-management.md)
ChunkMesh Vertex Packing and Management is a method for generating and maintaining dynamic triangle meshes from voxel data by packing vertex attributes into compact integer formats and managing buffer updates via a free-list allocator. It solves the problem of high memory bandwidth and GPU buffer upload overhead in voxel terrain rendering by minimizing vertex data size and enabling efficient incremental updates. It sets itself apart from sibling methods by focusing on the low-level data layout (bit-packing) and the specific logic for adding/removing faces between adjacent chunks, rather than just culling visibility or reindexing vertices.

### [Voxel Backface Culling and Internal Face Masking](4569-voxel-backface-culling-and-internal-face-masking.md)
This method optimizes voxel mesh rendering by rejecting faces that are either occluded by adjacent voxels (internal faces) or oriented away from the camera (backfaces). It distinguishes itself from standard neighbor-check culling by performing visibility tests in projection space to determine front/back orientation, while using a compact bitmask to track internal occlusion, allowing for efficient geometry shader-based culling.

### [Block Visibility Culling](5334-block-visibility-culling.md)
Block Visibility Culling is a coarse-grained optimization technique that determines which voxel blocks require mesh updates or rendering based on the camera's position and orientation, avoiding unnecessary geometry generation when the viewing direction has not changed significantly. Unlike sibling methods that operate at the fine-grained face level within a block (e.g., [Voxel Face Visibility Check](6723-voxel-face-visibility-check.md)), this method operates at the block level, using camera-relative coordinates to cull entire faces of a block before they are processed. It serves as a pre-filter to reduce the computational load of subsequent face-level culling and mesh assembly.

### [Extraction of Surface Points (Sobierajski)](5721-extraction-of-surface-points-sobierajski.md)
Extraction of Surface Points (Sobierajski) is a fast display method for the direct rendering of boundary surfaces by extracting only visible boundary voxels from volume data. It solves the problem of efficient surface visualization by projecting a pre-filtered set of 3D boundary voxels onto the image plane, rather than rendering the entire volume or a full mesh. It sets itself apart from sibling methods like [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md) by operating on a list of extracted surface points (voxels) rather than constructing a geometric mesh of faces, and from [Saito's Fast Previewing Algorithm](#saitos-fast-previewing-algorithm) by extracting the full visible boundary subset rather than a sparse uniform sample.

### [Mask Projection Culling](6836-mask-projection-culling.md)
Mask Projection Culling is a visibility optimization technique that filters mesh vertices and faces by projecting them into multiple camera views and checking if they fall within dilated binary masks, removing geometry not visible in any view. It sets itself apart from sibling methods like [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md) by relying on rasterization-space visibility (pixel coverage) rather than geometric neighbor checks, making it suitable for complex scenes where occlusion is determined by camera perspective rather than local voxel adjacency.

### [Mesh Culling via 3D Projection and Connected Components](6963-mesh-culling-via-3d-projection-and-connected-components.md)
This method culls mesh geometry by projecting vertices into multiple camera views and comparing their depths against reference depth maps to identify and remove unseen or artifact geometry. It distinguishes itself from neighbor-based culling (e.g., [Face Culling](8735-face-culling.md)) by relying on explicit visibility from camera poses rather than local voxel occupancy, making it suitable for unbounded surfaces or Gaussian splatting reconstructions where topological adjacency is not guaranteed.

### [Face Culling](8735-face-culling.md)
Face Culling is a mesh generation optimization for voxel-based volume rendering that reduces the number of vertices and triangles sent to the GPU by skipping the rendering of voxel faces that are adjacent to existing neighboring voxels. It solves the problem of naively rendering all 6 faces of every cube, including those hidden by neighbors, which results in excessive draw calls and vertex processing overhead. Unlike [Block Visibility Culling](5334-block-visibility-culling.md) which operates on entire blocks or [Opacity-based Mesh Culling](8759-opacity-based-mesh-culling.md) which considers material properties, Face Culling operates strictly on geometric adjacency within the voxel grid to eliminate internal faces.

### [Opacity-based Mesh Culling](8759-opacity-based-mesh-culling.md)
Opacity-based Mesh Culling is a post-processing technique that filters polygonal meshes extracted from volumetric or neural representations by discarding geometric primitives that contribute negligible visual opacity. It solves the problem of excessive mesh complexity and storage overhead in extracted surface representations. Unlike sibling methods that rely on voxel occupancy or spatial connectivity [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md), this method operates directly on the opacity values of intersection points on triangular faces, retaining only those meshes that meet a specific opacity threshold.

### [From-Region Visibility Preprocessing](9048-from-region-visibility-preprocessing.md)
From-region visibility preprocessing is a technique that precomputes visibility information for specific viewing regions to accelerate rendering of large datasets. It solves the problem of expensive per-frame visibility queries by shifting the computational burden to an offline or semi-offline phase. Unlike sibling methods such as [Face Culling](8735-face-culling.md) which rely on local geometric adjacency, this approach handles complex occlusion relationships, though it is distinguished by its significant preprocessing time requirements.

### Buried Bitmask Optimization
A bitmask used to track which blocks in a chunk are 'buried' (fully surrounded by opaque blocks), allowing the renderer to skip mesh generation for those blocks [S2, unofficial]. The process resets the buried bitmask for the chunk, creates `MappedAdjacentChunks` for the current chunk position to access neighboring chunks, and iterates through all blocks in the chunk (0 to `CHUNK_SIZE`) to determine if each is buried by calling `block->isBuried(...)` and setting the corresponding flag in the bitmask [S2, unofficial]. If all blocks in the chunk are buried (checked via `buriedBitmask->AreAllBlocksSet()`), the mesh is marked as empty and mesh generation is skipped [S2, unofficial]. When generating the mesh, any block that is air or has its buried flag set in the bitmask is skipped [S2, unofficial].

### Two-pass transparent meshing with opaque separation
A meshing strategy that separates opaque and transparent geometry into two passes to ensure correct rendering order and reduce overdraw, by generating opaque faces between air/transparent blocks and transparent faces only for blocks touching air [S3, unofficial]. The process first checks if the chunk contains any transparent blocks; if not, the second pass is skipped [S3, unofficial]. In the first (opaque) pass, if the chunk contains transparent blocks, it tests for opaque blocks (ignoring transparent and air) to generate opaque faces between air and transparent blocks; otherwise, it tests for non-air blocks [S3, unofficial]. In the second (transparent) pass, it tests for non-air blocks to mark only transparent blocks, generating faces for all transparent blocks touching air but not adding faces between transparent and solid blocks (since solid blocks already have those faces) [S3, unofficial]. To filter out solid block types during the transparent pass, before constructing the AO value and adding the block type to the hashmap, the code checks if it is in the transparent pass and ignores any block types that are not transparent [S3, unofficial].

## Background
The concept of removing interior polygons to improve rendering performance originated in the context of voxel-based procedural terrain generation, specifically referencing the implementation details found in Minecraft-like engines [S1, unofficial]. The initial observation was that a brute-force approach instantiating models of cubes resulted in the engine needlessly rendering 12 triangles (6 sides) for a cube where only 2 triangles (1 side) were visible [S1, unofficial]. To address this, a class was added to reconstruct cubes from triangles, allowing each cube to check for solid neighbors and determine which sides to render [S1, unofficial]. This optimization was further enhanced by combining the meshes of all individual cubes into a single big mesh called a 'chunk' [S1, unofficial].

## Sources

[S1] Voxel-Based Procedural Terrain Generation | PDF | Voxel | Rendering ..., https://www.scribd.com/document/840779393/Procedural-Terrain-Generation-Using-Voxels, fetched 2026-10-29 (unofficial)
[S2] gabkhanfig/Cube, https://github.com/gabkhanfig/Cube, fetched 2026-10-27 (unofficial)
[S3] Supporting Transparent Voxel Textures, https://github.com/TanTanDev/binary_greedy_mesher_demo/issues/5, fetched 2026-10-27 (unofficial)
