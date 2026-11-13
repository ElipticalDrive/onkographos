# Face Culling and Visibility Optimization

Also called: Interior Polygon Removal, Removing Interior Polygons

## Overview
Face Culling and Visibility Optimization is a family of mesh generation techniques that reduce rendering load by identifying and discarding geometric primitives (faces, vertices, or entire blocks) that are occluded by adjacent solid geometry or are not visible from the current camera viewpoint [S1, unofficial]. Unlike naive voxel rendering which generates all six faces for every occupied voxel, these methods skip the generation of internal faces, significantly reducing triangle count and GPU bandwidth usage [S1, unofficial]. This family is distinct from [Greedy face merging](8935-greedy-face-merging.md) because it focuses on *which* faces to generate based on visibility rules, whereas greedy merging focuses on *combining* adjacent visible faces into larger primitives. It also differs from [Voxel Block Placement with Face Culling](923-voxel-block-placement-with-face-culling.md) in that it encompasses a broader range of optimization strategies, including view-dependent culling and opacity-based filtering, rather than just basic neighbor checks.

## Shared principles
All methods in this family share the core principle of reducing the number of rendered primitives by avoiding the generation or transmission of geometry that does not contribute to the final image.

1.  **Occlusion Detection**: The primary mechanism is determining whether a face is blocked by a neighbor. In static culling, this is done by checking the occupancy state of adjacent voxel coordinates $P_{neighbor} = P + N$, where $P$ is the current voxel position and $N$ is the face normal [S1, S2, unofficial]. If the neighbor is solid, the face is internal and culled [S1, unofficial].
2.  **Visibility Pruning**: Methods extend neighbor checks to include view-dependent criteria. This includes backface culling (discarding faces with normals pointing away from the camera) and frustum culling (discarding geometry outside the view volume) [S1, S4, unofficial].
3.  **Batching and Chunking**: To optimize memory and draw calls, visible faces are often aggregated into larger structures called chunks or meshes. This requires handling boundary conditions between adjacent chunks to prevent rendering artifacts at chunk edges [S1, unofficial].
4.  **Transparency Handling**: For non-opaque blocks, culling logic must account for transparency. Simple neighbor checks may fail if the neighbor is transparent; thus, specialized culling modes (e.g., `TransparentMerged`, `TransparentUnmerged`) or opacity thresholds are used to decide whether to generate a face [S1, S3, unofficial].

## Subtopics

### [Voxel Block Shape & Culling Configuration](2176-voxel-block-shape-culling-configuration.md)
A trait-based system that decouples block identity from mesh generation logic. It allows different block types (e.g., opaque stone, transparent glass, sloped ramps) to define their own geometry and culling rules. It sets itself apart by supporting complex shapes (crosses, ramps) and fine-grained transparency merging via `TransparentGroup` identifiers, which is essential for fluids and non-colliding structures. Choose this when supporting diverse block types with specific geometric and lighting requirements.

### [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md)
Uses precomputed 16-bit bitmasks to represent the geometric coverage of voxel faces, allowing for the culling of partially occluded faces from partial shapes like stairs and slabs. It sets itself apart from basic neighbor checks by enabling fine-grained visibility decisions based on geometric overlap rather than simple adjacency. Choose this when rendering complex partial blocks where boolean solid/empty checks are insufficient.

### [ChunkMesh Vertex Packing and Management](3566-chunkmesh-vertex-packing-and-management.md)
A dynamic mesh generation technique that minimizes GPU memory bandwidth by packing vertex attributes into compact 32-bit integers and managing incremental updates via a free-list of reusable vertex slots. It sets itself apart by actively managing the vertex buffer lifecycle for dynamic worlds, supporting incremental updates without reallocation. Choose this for large, dynamic voxel worlds requiring frequent block updates.

### [Voxel Backface Culling and Internal Face Masking](4569-voxel-backface-culling-and-internal-face-masking.md)
Combines neighbor-based internal face masking with view-dependent backface culling in the vertex/geometry shader pipeline. It sets itself apart by using a compact bitmask and simplified projection-space heuristics to discard faces oriented away from the camera, reducing primitive count further than static culling alone. Choose this for real-time rendering where view-dependent optimization is critical.

### [Block Visibility Culling](5334-block-visibility-culling.md)
Operates at the coarse block level, determining which blocks require mesh regeneration based on camera position and rotation. It sets itself apart by caching visibility states and only recalculating face visibility when the camera crosses block boundaries or significantly changes orientation. Choose this for large-scale worlds to avoid redundant geometry processing for distant or static blocks.

### [Extraction of Surface Points (Sobierajski)](5721-extraction-of-surface-points-sobierajski.md)
A fast display method that extracts only visible boundary voxels as points, rather than generating a full mesh. It sets itself apart by operating on individual voxels projected to screen space and using multi-view visibility determination. Choose this for direct volume rendering or point-based visualization where mesh generation overhead is prohibitive.

### [Mask Projection Culling](6836-mask-projection-culling.md)
Removes faces and vertices not consistently observed across multiple camera poses by projecting 3D points into camera space and comparing depth against rendered depth maps or binary masks. It sets itself apart from neighbor-based culling by using actual camera visibility data to prune geometry, making it effective for removing floating artifacts and background noise in reconstructed meshes. Choose this for post-processing or reconstruction pipelines where multi-view consistency is available.

### [Mesh Culling via 3D Projection and Connected Components](6963-mesh-culling-via-3d-projection-and-connected-components.md)
Reduces mesh complexity by projecting vertices into multiple views, comparing depths against reference depth maps, and filtering by connected components. It sets itself apart from neighbor-based culling by relying on external camera viewpoints and from simple backface culling by using explicit depth testing. Choose this for cleaning up noisy meshes or removing occluded geometry in static scenes.

### [Face Culling](8735-face-culling.md)
The foundational optimization that prevents rendering of block faces occluded by adjacent solid voxels. It sets itself apart as a static, geometry-based optimization performed once during mesh creation, resulting in no framerate loss during rendering. Choose this as the baseline optimization for homogeneous voxel terrain.

### [Opacity-based Mesh Culling](8759-opacity-based-mesh-culling.md)
Discards triangles that do not contribute significantly to the final image based on local opacity values, often used with neural radiance fields. It sets itself apart by operating on radiance field parameters rather than voxel occupancy, enabling the conversion of neural fields into standard mesh formats. Choose this when working with implicit representations or learned opacity fields.

### [From-Region Visibility Preprocessing](9048-from-region-visibility-preprocessing.md)
Precomputes visibility relationships from specific viewpoints or regions to accelerate rendering. It sets itself apart by decoupling visibility determination from the runtime camera position, relying on global occlusion tests. Choose this for static scenes with fixed camera paths or offline baking.

### Buried Bitmask Optimization
A bitmask used to track which blocks in a chunk are 'buried' (fully surrounded by opaque blocks), allowing the renderer to skip mesh generation for those blocks. It sets itself apart by enabling chunk-level skipping based on aggregate occupancy. Choose this for large, static chunks where many blocks are internally occluded [S2, unofficial].

### Two-pass transparent meshing with opaque separation
A strategy that separates opaque and transparent geometry into two passes to ensure correct rendering order and reduce overdraw. It sets itself apart by generating opaque faces between air/transparent blocks first, then transparent faces only for blocks touching air. Choose this when rendering scenes with mixed opaque and transparent blocks to avoid transparency artifacts [S3, unofficial].

### Smart Leaf Rendering
A leaf block rendering strategy that preserves alpha cutouts on outer leaf surfaces while culling internal faces to reduce vertex count. It sets itself apart by specifically targeting foliage optimization. Choose this for terrain with dense vegetation [S4, unofficial].

### Hidden Face Extension
A proposed optimization where the 2D grid includes a 'don't care' state to allow rectangles to extend into hidden areas, potentially reducing total rectangle count. It sets itself apart by trading rasterizer load for mesh size reduction. Choose this with caution, as it may increase rasterizer load [S5, unofficial].

## Background
The concept of removing interior polygons was introduced in early voxel-based procedural terrain generation systems, such as those used in Minecraft-like engines. The initial observation was that naively rendering all six sides of every cube resulted in significant wasted rendering effort, as only the outermost faces were visible [S1, unofficial]. The first optimization involved reconstructing cubes from triangles and checking for solid neighbors to avoid rendering internal sides [S1, unofficial]. This was further optimized by combining meshes into chunks to reduce draw calls, although this introduced constraints on mesh size (e.g., Unity's 65,000 vertex limit) [S1, unofficial]. Over time, the family expanded to include view-dependent culling, bitmask optimizations for partial blocks, and opacity-based filtering for neural radiance fields [S2, S3, S1, unofficial].

## Sources

[S1] Voxel-Based Procedural Terrain Generation | PDF | Voxel | Rendering ..., https://www.scribd.com/document/840779393/Procedural-Terrain-Generation-Using-Voxels, fetched 2026-10-29 (unofficial)
[S2] gabkhanfig/Cube, https://github.com/gabkhanfig/Cube, fetched 2026-10-27 (unofficial)
[S3] Supporting Transparent Voxel Textures, https://github.com/TanTanDev/binary_greedy_mesher_demo/issues/5, fetched 2026-10-27 (unofficial)
[S4] ldominiq/ft_minecraft, https://github.com/ldominiq/ft_minecraft, fetched 2026-10-27 (unofficial)
[S5] Voxel Rendering Techniques - Medium, https://medium.com/@fogleman/voxel-rendering-techniques-fa8d869457ca, fetched 2026-10-27 (unofficial)
