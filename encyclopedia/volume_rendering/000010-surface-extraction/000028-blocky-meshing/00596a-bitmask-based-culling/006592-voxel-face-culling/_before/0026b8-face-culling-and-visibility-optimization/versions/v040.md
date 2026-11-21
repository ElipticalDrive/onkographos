# Face Culling and Visibility Optimization

Also called: Interior Polygon Removal, Removing Interior Polygons

## Overview
Face Culling and Visibility Optimization is a family of techniques used during or after voxel mesh generation to exclude geometric primitives that do not contribute to the final image, thereby reducing overdraw and rendering load [S1, unofficial]. It solves the problem of naively rendering all six faces of every solid cube, which results in massive overdraw in homogeneous terrain [S2, unofficial]. Unlike view-relative methods like [Voxel Backface Culling and Internal Face Masking](4569-voxel-backface-culling-and-internal-face-masking.md) which depend on camera orientation, standard Face Culling is static and determined solely by the local occupancy of the voxel grid [S1, S2, unofficial].

## Shared principles
The core mechanism across this family is the identification and elimination of redundant geometry. In the context of blocky meshing, a "face" is a polygonal surface (typically a quad composed of two triangles) belonging to a voxel block.

1.  **Topological Culling (Interior Polygon Removal):** The most fundamental optimization is checking neighbor occupancy. For a solid voxel at position $P$, a face in direction $d$ is culled if the neighbor at $P+d$ is also solid [S1, unofficial]. This prevents the rendering of internal faces shared between adjacent solid blocks. A single cube has 6 faces; in a dense terrain, most faces are internal. By removing these, the renderer only processes exposed surfaces [S1, unofficial].
2.  **View-Relative Culling:** Some methods extend topological culling by incorporating camera position. This includes back-face culling (rejecting faces whose normal points away from the camera) and frustum culling (rejecting blocks outside the view volume) [S4569].
3.  **Opacity-Based Culling:** For non-solid or translucent materials, culling may be based on material properties. Faces with opacity below a threshold $\tau_{prune}$ are discarded to reduce geometric complexity [S8759].
4.  **Mask-Based Culling:** For partial blocks (stairs, slabs), binary neighbor checks are insufficient. Bitmasks representing the geometric coverage of a face are used to determine if a neighbor fully occludes the visible regions of the current face [S3397].

## Subtopics

### [Voxel Block Placement with Face Culling](923-voxel-block-placement-with-face-culling.md)
A technique for dynamically updating a voxel chunk's internal state and face visibility when blocks are placed or removed. It uses neighbor checks to toggle shared faces without regenerating the entire mesh. It differs from [Greedy face merging](8935-greedy-face-merging.md) by maintaining per-block visibility flags rather than merging adjacent quads into larger primitives, and from [Blocky meshing](31-blocky-meshing.md) generally by focusing on incremental state updates rather than batch mesh generation.

### [Voxel Block Shape & Culling Configuration](2176-voxel-block-shape-culling-configuration.md)
A trait-based system that defines the geometry, transparency culling behavior, and lighting properties for individual block types. It solves the problem of hard-coded geometry by decoupling block identity from mesh generation, allowing different block types (e.g., opaque stone, transparent glass) to specify their own rendering characteristics. Unlike generic mesh generation that treats all voxels as cubes, this configuration allows for complex shapes and fine-grained culling modes (such as merging transparent faces) to be defined per block type.

### [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md)
A method for optimizing blocky mesh generation by determining face visibility based on partial geometric coverage rather than binary solidity. It solves the problem where standard neighbor-based culling fails for partial shapes (such as stairs, slabs, or leaves), which have mixed visible and hidden areas on a single face boundary. Unlike sibling methods that treat faces as fully solid or fully empty, this method uses precomputed bitmasks to represent the occupancy of a face's sub-regions, allowing for more accurate culling of occluded geometry while maintaining high performance through bitwise operations. It is distinct from [Mask Projection Culling](6836-mask-projection-culling.md) because it operates in local block space using precomputed static masks rather than runtime camera projections, and from [Opacity-based Mesh Culling](8759-opacity-based-mesh-culling.md) because it handles geometric coverage independently of material transparency, though it can be combined with it.

### [ChunkMesh Vertex Packing and Management](3566-chunkmesh-vertex-packing-and-management.md)
A dynamic mesh generation and buffer management technique for voxel terrain, designed to minimize GPU memory bandwidth and draw calls by tightly packing vertex attributes into integer buffers and efficiently handling incremental mesh updates. It solves the problem of sparse, high-overhead vertex buffers in blocky meshing by encoding position, texture, lighting, and texture coordinates into two 32-bit integers per vertex, while maintaining a free-list of reusable vertex indices to avoid reallocation. It sets itself apart from static meshing methods by supporting incremental add/remove operations with neighbor-aware face culling, and from pure GPU-based culling by performing visibility logic and buffer compaction on the CPU side before uploading to the GPU.

### [Voxel Backface Culling and Internal Face Masking](4569-voxel-backface-culling-and-internal-face-masking.md)
This method rejects invisible faces in voxel rendering by determining front/back orientation in projection space and masking faces shared between neighboring voxels. It sets itself apart from simple topological culling by using view-dependent projection space tests to determine face visibility, while simultaneously using bitmask flags to exclude internal faces that are occluded by adjacent voxels.

### [Block Visibility Culling](5334-block-visibility-culling.md)
Block Visibility Culling determines which voxel blocks require mesh regeneration or rendering updates based on the camera's position and orientation, avoiding unnecessary geometry processing when the viewer has not moved significantly or changed their primary viewing direction. It operates at the coarse block level by comparing current camera coordinates and viewing direction against previous frame values, serving as a high-level optimization that complements finer-grained face culling. Unlike [Voxel Backface Culling and Internal Face Masking](4569-voxel-backface-culling-and-internal-face-masking.md), which evaluates individual surface normals, this method evaluates the visibility of entire blocks based on camera-relative orientation.

### [Extraction of Surface Points (Sobierajski)](5721-extraction-of-surface-points-sobierajski.md)
Extraction of Surface Points is a direct volume rendering method that accelerates surface visualization by extracting only the visible boundary voxels from the volume data, rather than generating a full mesh or ray-casting the entire volume. It solves the problem of high computational cost in surface rendering by reducing the number of primitives sent to the rasterizer. It differs from [Face Culling](8735-face-culling.md) by operating on individual voxels and projecting them as points rather than generating polygonal faces, and from [Voxel Block Placement with Face Culling](923-voxel-block-placement-with-face-culling.md) by using a multi-view visibility check to determine which voxels are potentially visible before projection.

### [Mask Projection Culling](6836-mask-projection-culling.md)
Mask Projection Culling is a visibility-based mesh filtering technique that removes faces and vertices not consistently observed across multiple camera poses by projecting points into camera space and comparing their depth against rendered depth maps. It differs from topological culling (which relies on voxel adjacency) and simple back-face culling (which relies on normal orientation) by using multi-view geometric consistency to identify and discard occluded or background geometry. It builds on [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md) by implementing the "View-Relative Culling" category through explicit projection and depth testing.

### [Mesh Culling via 3D Projection and Connected Components](6963-mesh-culling-via-3d-projection-and-connected-components.md)
This method removes geometric artifacts and occluded geometry from unbounded meshes (typically derived from Gaussian Splatting) by projecting vertices into multiple camera views and validating visibility against reference depth maps. It distinguishes itself from topological culling by using view-dependent depth consistency rather than voxel neighbor occupancy, and from simple back-face culling by requiring multi-view agreement. It serves as a robust post-processing step for mesh extraction from radiance fields, specifically targeting the "halo" artifacts common in unbounded reconstructions.

### [Face Culling](8735-face-culling.md)
Face Culling is a topological optimization used during voxel mesh generation to exclude geometric primitives that do not contribute to the final image by identifying faces occluded by adjacent solid voxels. It solves the problem of naively rendering all six faces of every solid cube, which results in massive overdraw and poor performance in homogeneous terrain. Unlike view-relative methods like [Voxel Backface Culling and Internal Face Masking](4569-voxel-backface-culling-and-internal-face-masking.md) which depend on camera orientation, Face Culling is static and determined solely by the local occupancy of the voxel grid.

### [Opacity-based Mesh Culling](8759-opacity-based-mesh-culling.md)
Opacity-based Mesh Culling is a post-processing optimization technique used to reduce the geometric complexity of extracted polygonal meshes by discarding faces that contribute negligible visual information. It operates by evaluating the opacity (or density) of mesh faces against a predefined threshold, retaining only those that are sufficiently opaque to impact the final image. This method distinguishes itself from topological culling, which relies on voxel neighbor occupancy, and view-relative culling, which depends on camera orientation, by focusing purely on the material property (opacity) of the geometry itself. It is primarily used to reduce storage costs and improve rasterization performance by decreasing the triangle count, often resulting in a significant reduction (e.g., to 5% of the original count).

### [From-Region Visibility Preprocessing](9048-from-region-visibility-preprocessing.md)
From-region visibility preprocessing is a technique that precomputes which geometric primitives are visible from specific regions of the viewing space, allowing for the culling of occluded geometry during rendering. It differs from view-relative culling methods like [Voxel Backface Culling and Internal Face Masking](4569-voxel-backface-culling-and-internal-face-masking.md) by being independent of the camera's instantaneous position, and from topological culling like [Face Culling](8735-face-culling.md) by handling complex, non-voxel-based geometry where neighbor relationships are not sufficient to determine occlusion.

### Buried Bitmask Optimization
A bitmask used to track which blocks in a chunk are 'buried' (fully surrounded by opaque blocks), allowing the renderer to skip mesh generation for those blocks [S2, unofficial]. The process resets the buried bitmask for the chunk, creates mapped adjacent chunks for neighbor access, and iterates through all blocks to determine if they are buried. If all blocks in a chunk are buried, the mesh is marked as empty and generation is skipped. During mesh generation, any block that is air or has its buried flag set is skipped [S2, unofficial].

### Two-pass transparent meshing with opaque separation
A meshing strategy that separates opaque and transparent geometry into two passes to ensure correct rendering order and reduce overdraw, by generating opaque faces between air/transparent blocks and transparent faces only for blocks touching air [S3, unofficial]. The first pass (opaque) generates faces for opaque blocks, testing for opaque blocks if transparent blocks exist in the chunk (ignoring transparent and air), or non-air blocks otherwise. This generates opaque faces between air and transparent blocks. The second pass (transparent) tests for non-air blocks to mark only transparent blocks, generating faces for all transparent blocks touching air but not adding faces between transparent and solid blocks. Solid block types are filtered out during the transparent pass [S3, unofficial].

### Smart Leaf Rendering
A leaf block rendering strategy that preserves alpha cutouts on outer leaf surfaces while culling internal faces to reduce vertex count [S4, unofficial]. It is controlled by a global static enum `sLeafRenderMode`, which defaults to `ChunkRenderer::LeafRenderMode::Smart` [S4, unofficial].

### Hidden Face Extension
A proposed optimization where the 2D grid includes a 'don't care' state (2) to allow rectangles to extend into hidden areas, potentially reducing total rectangle count [S5, unofficial]. While this might reduce mesh size, it increases the load on the rasterizer [S5, unofficial].

## Background
The concept of removing interior polygons dates back to early voxel-based terrain generation, such as in Minecraft-like engines. In these systems, a brute-force approach of instantiating individual cube models was replaced by reconstructing cubes from triangles and removing sides adjacent to solid neighbors [S1, unofficial]. This optimization was critical because rendering all 12 triangles (6 sides) for every cube was inefficient when only 2 triangles (1 side) were visible [S1, unofficial]. The meshes of individual cubes were combined into larger "chunks" to further optimize implementation, though this introduced constraints such as vertex limits in early graphics APIs like Unity's [S1, unofficial].

## Sources

[S1] Voxel-Based Procedural Terrain Generation | PDF | Voxel | Rendering ..., https://www.scribd.com/document/840779393/Procedural-Terrain-Generation-Using-Voxels, fetched 2026-10-29 (unofficial)
[S2] gabkhanfig/Cube, https://github.com/gabkhanfig/Cube, fetched 2026-10-27 (unofficial)
[S3] Supporting Transparent Voxel Textures, https://github.com/TanTanDev/binary_greedy_mesher_demo/issues/5, fetched 2026-10-27 (unofficial)
[S4] ldominiq/ft_minecraft, https://github.com/ldominiq/ft_minecraft, fetched 2026-10-27 (unofficial)
[S5] Voxel Rendering Techniques - Medium, https://medium.com/@fogleman/voxel-rendering-techniques-fa8d869457ca, fetched 2026-10-27 (unofficial)
