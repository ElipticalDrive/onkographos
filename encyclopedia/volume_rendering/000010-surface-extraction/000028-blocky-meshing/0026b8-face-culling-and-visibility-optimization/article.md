# Face Culling and Visibility Optimization

Also called: Interior Polygon Removal, Removing Interior Polygons

## Overview
Face Culling and Visibility Optimization is a family of techniques that reduce the geometric complexity of voxel-based scenes by identifying and discarding faces that are occluded by adjacent solid voxels or invisible to the camera. It solves the problem of naive mesh generation, where every voxel contributes six faces (12 triangles), resulting in excessive triangle counts and rendering overhead for internal geometry [S1, unofficial]. Unlike sibling methods that focus on merging adjacent faces into larger primitives ([Greedy face merging](8935-greedy-face-merging.md)) or dynamic block placement ([Voxel Block Placement with Face Culling](923-voxel-block-placement-with-face-culling.md)), this family primarily addresses the static or incremental removal of hidden surfaces to minimize vertex buffer size and draw calls [S1, S2, unofficial].

## Shared principles
The core mechanism across this family is the determination of face visibility based on local neighborhood occupancy or global view frustum constraints.

1.  **Neighbor Check**: For a voxel at position $P$ with a face normal $N$, the neighbor position is calculated as $P_{neighbor} = P + N$ [S1, S2, unofficial].
2.  **Occupancy Test**: The system checks if the voxel at $P_{neighbor}$ is solid (non-air).
3.  **Culling Decision**:
    *   If the neighbor is solid, the face is internal and culled (not rendered) [S1, S2, unofficial].
    *   If the neighbor is empty (air) or out of bounds, the face is exposed and rendered [S2, unofficial].
4.  **Bottom Face Culling**: In many implementations, the bottom face (normal $-Y$) is always culled or hidden, as it is typically occluded by the ground or not visible from standard viewing angles [S2, unofficial].
5.  **Batching**: Individual cube meshes are often combined into a single "chunk" mesh to optimize GPU draw calls, subject to vertex count limits (e.g., ~65,000 vertices in Unity) [S1, unofficial].

## Subtopics

### [Voxel Block Placement with Face Culling](923-voxel-block-placement-with-face-culling.md)
A technique for dynamically updating a voxel chunk's internal state and face visibility when blocks are placed or removed, using neighbor checks to toggle shared faces without regenerating the entire mesh. It differs from its sibling [Greedy face merging](8935-greedy-face-merging.md) by maintaining per-block visibility flags rather than merging adjacent quads into larger primitives, and from [Blocky meshing](31-blocky-meshing.md) generally by focusing on incremental state updates rather than batch mesh generation.

### [Voxel Block Shape & Culling Configuration](2176-voxel-block-shape-culling-configuration.md)
A trait-based system that defines block geometry, transparency culling modes, and lighting properties per block type. It solves the problem of heterogeneous block behavior in voxel worlds by decoupling mesh generation logic from the core culling algorithm. Unlike sibling methods that assume uniform block geometry or simple boolean transparency, this approach allows distinct culling strategies (e.g., merging transparent faces vs. keeping them separate) and complex shapes (e.g., ramps, fluids) to be configured via a single interface.

### [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md)
A method for optimizing blocky mesh generation by representing the geometric coverage of a voxel face as a bit mask, allowing for the culling of partially occluded faces from partial shapes (such as stairs and slabs). Unlike basic neighbor checks that treat faces as binary solid/empty, this method solves the problem where a boolean solid flag is insufficient because partial shapes have mixed visible and hidden areas on a single face. It sets itself apart from [Face Culling](8735-face-culling.md) by enabling fine-grained visibility decisions based on geometric overlap rather than simple adjacency, and from [Opacity-based Mesh Culling](8759-opacity-based-mesh-culling.md) by separating geometric coverage from material transparency properties.

### [ChunkMesh Vertex Packing and Management](3566-chunkmesh-vertex-packing-and-management.md)
A dynamic mesh generation and buffer management technique for blocky voxel terrain, designed to minimize GPU memory bandwidth and draw call overhead by packing vertex attributes into compact integer formats and managing vertex lifecycles via a free-list allocator. It solves the problem of sparse, fragmented vertex buffers in incremental voxel meshing by reusing vertex slots and compacting data before upload. Unlike static meshing methods, it supports real-time addition and removal of blocks with immediate neighbor face updates, ensuring consistency across chunk boundaries.

### [Voxel Backface Culling and Internal Face Masking](4569-voxel-backface-culling-and-internal-face-masking.md)
A technique to reject invisible faces in voxel rendering by determining front/back orientation in projection space and masking faces shared between neighboring voxels. It sets itself apart from sibling methods by performing visibility checks directly within the geometry shader pipeline using a packed 32-bit integer attribute, allowing for immediate rejection of internal faces and back-facing quads without requiring pre-computed mesh topology or complex chunk boundary handling.

### [Block Visibility Culling](5334-block-visibility-culling.md)
Determines which voxel blocks require mesh regeneration or rendering updates based on the camera's position and orientation, avoiding unnecessary geometry processing when the viewing direction or block-relative position remains stable. It solves the problem of redundant computation in dynamic voxel scenes by skipping index updates if the camera has not moved significantly or changed its primary viewing direction relative to the block grid. Unlike sibling methods that operate at the fine-grained face level within a block [6723-voxel-face-visibility-check.md], this method operates at the coarse block level, using camera-relative coordinates and surface orientation flags to prune entire block faces before they are submitted to the rasterizer.

### [Extraction of Surface Points (Sobierajski)](5721-extraction-of-surface-points-sobierajski.md)
A fast display method for the direct rendering of boundary surfaces by extracting only visible boundary voxels from volume data. It solves the problem of high computational cost in volume rendering by avoiding the processing of internal or occluded geometry. It sets itself apart from sibling methods like [Voxel Block Placement with Face Culling](923-voxel-block-placement-with-face-culling.md) by operating on individual voxel points rather than blocky meshes, and from [Face Culling](8735-face-culling.md) by using a pre-extraction visibility filter based on orthographic views rather than per-face geometric checks during rasterization.

### [Mask Projection Culling](6836-mask-projection-culling.md)
A visibility-based mesh filtering technique that removes faces and vertices not consistently observed across multiple camera poses by projecting points into camera space and comparing their depth against rendered depth maps. It solves the problem of retaining only the geometry visible from a specific set of viewpoints, effectively pruning occluded or background geometry. Unlike static neighbor-based culling in [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md), this method is view-dependent and relies on multi-view consistency rather than local voxel occupancy. It differs from [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md) by explicitly using depth comparison to determine front-facing visibility, whereas coverage masks may rely solely on 2D projection overlap.

### [Mesh Culling via 3D Projection and Connected Components](6963-mesh-culling-via-3d-projection-and-connected-components.md)
This method reduces mesh complexity in unbounded Gaussian splatting reconstructions by classifying 3D points as seen, unseen, or forecast, and culling mesh geometry by comparing projected point depths against reference depth maps from multiple camera poses. It sets itself apart from sibling methods like [Face Culling](8735-face-culling.md) by using multi-view consistency rather than local voxel adjacency, and from [Block Visibility Culling](5334-block-visibility-culling.md) by operating on the final triangle mesh rather than the underlying voxel grid. It specifically targets artifacts arising from unbounded reconstruction where distant or occluded geometry may persist.

### [Face Culling](8735-face-culling.md)
Face Culling is a mesh generation optimization that prevents the rendering of block faces occluded by adjacent solid voxels, significantly reducing triangle count in homogeneous voxel terrain. It solves the problem of naively rendering all six faces of every cube, including those hidden by neighbors, by skipping the generation of faces where the adjacent voxel coordinate is also solid. Unlike [Voxel Backface Culling and Internal Face Masking](4569-voxel-backface-culling-and-internal-face-masking.md) which operates on the GPU or during rasterization, this method performs static occlusion detection during mesh creation to minimize vertex buffer size and draw calls.

### [Opacity-based Mesh Culling](8759-opacity-based-mesh-culling.md)
Opacity-based Mesh Culling reduces the geometric complexity of a scene by discarding or clipping triangular faces that contribute negligible visual information, specifically those with low accumulated opacity. It solves the problem of excessive mesh size in neural radiance field (NeRF) or implicit surface representations by retaining only meshes that meet a specific opacity criterion, thereby reducing mesh count and improving rendering speed. Unlike sibling methods that rely on voxel occupancy or spatial block visibility, this method operates directly on the parametric triangle representation, filtering based on material opacity rather than geometric occupancy.

### [From-Region Visibility Preprocessing](9048-from-region-visibility-preprocessing.md)
From-region visibility preprocessing is a technique that precomputes occlusion relationships for specific viewing regions to avoid rendering primitives that are blocked by others. Unlike static culling methods that rely on local neighbor checks, this approach determines global visibility from a region's perspective, often using occluder fusion to simplify the occlusion geometry. It sets itself apart from its siblings by shifting the computational burden to an offline preprocessing phase to enable faster runtime rendering, though this comes at the cost of significant preprocessing time.

### Buried Bitmask Optimization
A bitmask used to track which blocks in a chunk are 'buried' (fully surrounded by opaque blocks), allowing the renderer to skip mesh generation for those blocks [S2, unofficial]. The process involves resetting the buried bitmask for the chunk, creating mapped adjacent chunks for neighbor access, and iterating through all blocks to determine if they are buried. If all blocks in a chunk are buried, the mesh is marked as empty and generation is skipped. During mesh generation, any block that is air or has its buried flag set is skipped [S2, unofficial].

### Two-pass transparent meshing with opaque separation
A meshing strategy that separates opaque and transparent geometry into two passes to ensure correct rendering order and reduce overdraw, by generating opaque faces between air/transparent blocks and transparent faces only for blocks touching air [S3, unofficial]. The first pass generates opaque faces, testing for opaque blocks (ignoring transparent and air if transparent blocks exist) to create faces between air and transparent blocks. The second pass tests for non-air blocks to mark only transparent blocks, generating faces for transparent blocks touching air but not between transparent and solid blocks. Solid block types are filtered out in the transparent pass to avoid duplicate geometry [S3, unofficial].

### Smart Leaf Rendering
A leaf block rendering strategy that preserves alpha cutouts on outer leaf surfaces while culling internal faces to reduce vertex count [S4, unofficial]. It is controlled by a global static enum `sLeafRenderMode`, defaulting to `ChunkRenderer::LeafRenderMode::Smart` [S4, unofficial].

### Hidden Face Extension
A proposed optimization where the 2D grid includes a 'don't care' state (2) to allow rectangles to extend into hidden areas, potentially reducing total rectangle count [S5, unofficial]. While this might reduce mesh size, it increases the load on the rasterizer [S5, unofficial].

## Background
The concept of removing interior polygons was introduced in early voxel terrain generation systems, such as those described in "Voxel-Based Procedural Terrain Generation" [S1, unofficial]. The author noted that naively rendering cubes resulted in 12 triangles per side (6 sides), whereas only 2 triangles (1 side) were typically visible. By reconstructing cubes from triangles and checking for solid neighbors, the engine could remove internal sides, significantly improving performance. The meshes of individual cubes were then combined into "chunks" to further optimize implementation, though this introduced vertex count limits (e.g., ~65,000 vertices in Unity) [S1, unofficial].

Later implementations, such as the "Cube" library [S2, unofficial], formalized the neighbor-checking algorithm, using hash sets or boolean arrays to track occupied positions and determine face visibility. The "Cube" method also introduced bitmask optimizations for buried blocks [S2, unofficial].

Handling transparency required more complex strategies. The "binary_greedy_mesher_demo" project [S3, unofficial] highlighted the need for two-pass meshing to separate opaque and transparent geometry, ensuring correct rendering order and minimizing overdraw. This approach involves generating opaque faces between air and transparent blocks in the first pass, and transparent faces only for blocks touching air in the second pass [S3, unofficial].

Advanced techniques like "Face Coverage Mask Culling" [S2, unofficial] and "Mask Projection Culling" [S1, S2, unofficial] extended basic neighbor checks to handle partial shapes (stairs, slabs) and view-dependent culling using depth maps and bitmasks. Sobierajski's method [S1, unofficial] applied similar visibility principles to volume rendering, extracting only visible boundary voxels using orthographic views.

More recent developments in neural rendering and Gaussian splatting have adapted these principles. "Opacity-based Mesh Culling" [S1, S2, unofficial] filters triangles based on learned opacity parameters, while "Mesh Culling via 3D Projection" [S1, unofficial] uses multi-view consistency to prune geometry in unbounded reconstructions.

## Sources

[S1] Voxel-Based Procedural Terrain Generation | PDF | Voxel | Rendering ..., https://www.scribd.com/document/840779393/Procedural-Terrain-Generation-Using-Voxels, fetched 2026-10-29 (unofficial)
[S2] gabkhanfig/Cube, https://github.com/gabkhanfig/Cube, fetched 2026-10-27 (unofficial)
[S3] Supporting Transparent Voxel Textures, https://github.com/TanTanDev/binary_greedy_mesher_demo/issues/5, fetched 2026-10-27 (unofficial)
[S4] ldominiq/ft_minecraft, https://github.com/ldominiq/ft_minecraft, fetched 2026-10-27 (unofficial)
[S5] Voxel Rendering Techniques - Medium, https://medium.com/@fogleman/voxel-rendering-techniques-fa8d869457ca, fetched 2026-10-27 (unofficial)
