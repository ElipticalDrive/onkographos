# Face Culling and Visibility Optimization

Also called: Interior Polygon Removal, Removing Interior Polygons

## Overview
Face Culling and Visibility Optimization reduces the rendering load of voxel meshes by reconstructing cubes from triangles and removing sides that are adjacent to solid neighbors, thereby avoiding the rendering of 12 triangles (6 sides) when only 2 triangles (1 side) are visible [S1, unofficial]. It solves the problem of naively rendering all 6 faces of every cube, including those hidden by neighbors. It sets itself apart from its siblings by relying on direct local topology checks (neighbor existence) rather than coverage masks, backface culling, or global visibility preprocessing [8735-face-culling.md].

## Shared principles
The core mechanism involves iterating through voxel positions and determining face visibility based on the occupancy state of immediate neighbors. A common implementation uses a hash set or grid lookup for $O(1)$ neighbor checks [8735-face-culling.md].

For each voxel at position $P$, the algorithm evaluates the 6 potential faces. For a face with normal $N_{face}$, the neighbor position is calculated as:
$$
P_{neighbor} = P + N_{face}
$$
The face is culled if a solid voxel exists at $P_{neighbor}$ [S1, unofficial].

In standard implementations, bottom faces are often always culled to hide the underside of terrain, regardless of whether there is a voxel below [8735-face-culling.md]. Side faces are culled if a neighbor exists directly adjacent or if a neighbor exists directly above the neighbor position (to handle overhangs) [8735-face-culling.md].

## Subtopics

### [Voxel Block Shape & Culling Configuration](2176-voxel-block-shape-culling-configuration.md)
Voxel Block Shape & Culling Configuration is a trait-based system that defines the geometry, transparency culling behavior, and lighting properties for each block type in a voxel mesh generator. It solves the problem of rigid, single-shape meshing by allowing distinct block types (e.g., glass, water, ramps) to specify their own vertex layouts and face-culling rules. Unlike its sibling [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md), which relies on a generic boolean occupancy check for axis-aligned cubes, this method decouples block identity from geometry, enabling complex shapes and specialized rendering passes (such as fluid stitching or alpha blending) without modifying the core meshing loop.

### [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md)
Face Coverage Mask Culling is a method for removing hidden faces of partial voxel shapes (such as stairs, slabs, or custom geometries) by representing face coverage as a bit mask. It solves the problem where a boolean "solid/empty" flag is insufficient because partial shapes have mixed visible and hidden areas on a single face. Unlike sibling methods that rely on coarse block visibility or vertex packing, this method operates at the sub-face granularity, allowing precise culling of geometry that is occluded by neighbors while preserving exposed portions of the same face.

### [ChunkMesh Vertex Packing and Management](3566-chunkmesh-vertex-packing-and-management.md)
ChunkMesh Vertex Packing and Management is a dynamic mesh generation and update technique for voxel terrain that minimizes GPU memory bandwidth and buffer upload overhead by packing vertex attributes into two 32-bit unsigned integers per vertex. It solves the problem of sparse, fragmented vertex buffers in incremental blocky meshing by reusing vertex slots and compacting data in-place. It sets itself apart from sibling methods like [Voxel Block Placement with Face Culling](923-voxel-block-placement-with-face-culling.md) by focusing on the internal data structure and memory management of the mesh itself, rather than just the visibility logic. Unlike [Voxel Face Meshing with Packed Attributes](4153-voxel-face-meshing-with-packed-attributes.md), which describes the shader-side unpacking, this method details the CPU-side generation, packing, and dynamic buffer update loop.

### [Voxel Backface Culling and Internal Face Masking](4569-voxel-backface-culling-and-internal-face-masking.md)
This method optimizes voxel mesh rendering by rejecting faces that are either occluded by adjacent voxels (internal faces) or oriented away from the camera (backfaces). It distinguishes itself from standard neighbor-check culling by performing visibility tests in projection space to determine front/back orientation, while using a compact bitmask to track internal occlusion, allowing for efficient geometry shader-based culling.

### [Block Visibility Culling](5334-block-visibility-culling.md)
Block Visibility Culling determines which voxel blocks require mesh updates or rendering based on the camera's position and rotation, avoiding unnecessary geometry generation when the viewing direction or block-relative position has not changed significantly. Unlike its parent [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md), which relies on local voxel topology (neighbor occupancy) to cull faces, this method operates at the coarse block level using camera-relative geometric checks to skip entire blocks or specific surface orientations. It is distinct from [Voxel Face Visibility Check](6723-voxel-face-visibility-check.md) by operating on the block aggregate rather than individual faces within a block.

### [Extraction of Surface Points (Sobierajski)](5721-extraction-of-surface-points-sobierajski.md)
A fast display method for direct rendering of boundary surfaces by extracting only visible boundary voxels from volume data. It sets itself apart from sibling methods like [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md) by operating on a per-voxel basis using global viewing directions rather than local neighbor checks, and from [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md) by relying on 2D boundary tracing rather than coverage masks.

### [Mask Projection Culling](6836-mask-projection-culling.md)
Mask Projection Culling is a visibility-based mesh filtering technique that removes faces and vertices not consistently observed across multiple camera poses by projecting points into camera space and comparing their depth against rendered depth maps or binary masks. It differs from local topology-based methods like [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md) by relying on global camera viewpoints rather than immediate voxel neighbors, allowing it to remove geometry occluded by distant objects or outside the field of view. It is distinct from [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md) which typically uses pre-computed coverage masks, whereas this method actively renders depth maps or samples existing masks during the culling process.

### [Mesh Culling via 3D Projection and Connected Components](6963-mesh-culling-via-3d-projection-and-connected-components.md)
This method culls mesh geometry by projecting vertices into multiple camera views and comparing their depths against reference depth maps to classify faces as seen, unseen, or forecast. It sets itself apart from local topology-based siblings like [Face Culling](8735-face-culling.md) by using global visibility information from multiple viewpoints, making it effective for removing artifacts in unbounded reconstructions.

### [Face Culling](8735-face-culling.md)
Face Culling is a mesh generation optimization that reduces the number of vertices sent to the GPU by skipping the rendering of voxel faces that are adjacent to existing neighboring voxels. It solves the problem of naively rendering all 6 faces of every cube, including those hidden by neighbors. It sets itself apart from its siblings by relying on direct local topology checks (neighbor existence) rather than coverage masks, backface culling, or global visibility preprocessing.

### [Opacity-based Mesh Culling](8759-opacity-based-mesh-culling.md)
Opacity-based Mesh Culling is a post-processing technique for extracting compact polygonal meshes from volumetric or implicit representations by discarding geometric primitives that contribute negligible visual weight. It solves the problem of excessive mesh complexity in neural radiance field (NeRF) or similar implicit rendering pipelines, where the initial output often contains redundant geometry. Unlike its siblings, which rely on topological voxel occupancy [923-voxel-block-placement-with-face-culling.md] or spatial projection [6963-mesh-culling-via-3d-projection-and-connected-components.md], this method evaluates the radiometric opacity of individual triangles or faces to determine visibility, retaining only those that significantly affect the final image.

### [From-Region Visibility Preprocessing](9048-from-region-visibility-preprocessing.md)
From-region visibility preprocessing is a technique for optimizing the rendering of large-scale 3D environments by precomputing which geometric primitives are visible from specific regions of the scene. It solves the problem of expensive per-frame visibility culling for massive datasets by shifting the computational burden to an offline phase. It is distinguished from its siblings, such as [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md), which rely on local voxel topology and axis-aligned assumptions, by operating on general geometry (triangles) and using occluder fusion to reduce the number of visible primitives, albeit at the cost of significantly longer preprocessing times.

### Buried Bitmask Optimization
A bitmask used to track which blocks in a chunk are 'buried' (fully surrounded by opaque blocks), allowing the renderer to skip mesh generation for those blocks [S2, unofficial]. The algorithm resets the buried bitmask for the chunk, creates `MappedAdjacentChunks` for the current chunk position to access neighboring chunks, and iterates through all blocks in the chunk (0 to `CHUNK_SIZE`). For each block, it determines if it is buried by calling `block->isBuried(...)` and sets the corresponding flag in the bitmask. If all blocks in the chunk are buried (checked via `buriedBitmask->AreAllBlocksSet()`), the mesh is marked as empty and mesh generation is skipped. When generating the mesh, any block that is air or has its buried flag set in the bitmask is skipped [S2, unofficial].

### Two-pass transparent meshing with opaque separation
A meshing strategy that separates opaque and transparent geometry into two passes to ensure correct rendering order and reduce overdraw, by generating opaque faces between air/transparent blocks and transparent faces only for blocks touching air [S3, unofficial]. The algorithm first checks if the chunk contains any transparent blocks; if not, it skips the second pass entirely. In the first (opaque) pass, if the chunk contains transparent blocks, it tests for opaque blocks (ignoring transparent and air) to generate opaque faces between air and transparent blocks; otherwise, it tests for non-air blocks. In the second (transparent) pass, it tests for non-air blocks to mark only transparent blocks, generating faces for all transparent blocks touching air but not adding faces between transparent and solid blocks. To filter out solid block types during the transparent pass, before constructing the AO value and adding the block type to the hashmap, it checks if it is in the transparent pass and ignores any block types that are not transparent [S3, unofficial].

### Smart Leaf Rendering
A leaf block rendering strategy that preserves alpha cutouts on outer leaf surfaces while culling internal faces to reduce vertex count [S4, unofficial]. It is controlled by a global static enum `sLeafRenderMode`, which defaults to `ChunkRenderer::LeafRenderMode::Smart` [S4, unofficial].

## Background
The technique originated with early voxel engines like Minecraft, where the brute-force approach of instantiating models for every cube was replaced by reconstructing cubes from triangles and removing inner polygons to improve rendering performance [S1, unofficial]. Each cube consists of 12 triangles (2 of which make a cube’s side); initially, the engine was needlessly rendering 12 triangles (6 sides) while only 2 triangles (1 side) were visible [S1, unofficial]. To replace the brute-force approach, a class that creates cubes from triangles was added to allow each cube to check whether it has a solid neighbor and determine which sides of itself it should render [S1, unofficial]. To further optimize, the meshes of all cubes were combined into a single big mesh called a 'chunk' [S1, unofficial].

## Sources
[S1] Voxel-Based Procedural Terrain Generation | PDF | Voxel | Rendering ..., https://www.scribd.com/document/840779393/Procedural-Terrain-Generation-Using-Voxels, fetched 2026-10-29 (unofficial)
[S2] gabkhanfig/Cube, https://github.com/gabkhanfig/Cube, fetched 2026-10-27 (unofficial)
[S3] Supporting Transparent Voxel Textures, https://github.com/TanTanDev/binary_greedy_mesher_demo/issues/5, fetched 2026-10-27 (unofficial)
[S4] ldominiq/ft_minecraft, https://github.com/ldominiq/ft_minecraft, fetched 2026-10-27 (unofficial)
