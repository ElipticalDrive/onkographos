# Face Culling and Visibility Optimization

Also called: Interior Polygon Removal, Removing Interior Polygons

## Overview
Face Culling and Visibility Optimization is a family of techniques that reduce the rendering load of voxel-based volume rendering by identifying and discarding geometric primitives (faces, vertices, or entire blocks) that are occluded by neighbors, oriented away from the camera, or otherwise invisible. It solves the problem of naive mesh generation, which typically produces 12 triangles (6 sides) per voxel even when only 2 triangles (1 side) are visible, leading to excessive overdraw and GPU bandwidth waste [S1, unofficial]. This family is distinguished from general surface extraction methods by its focus on immediate geometric adjacency checks, camera-space orientation tests, or precomputed visibility masks to prune geometry before or during mesh assembly.

## Shared principles
All methods in this family operate on the principle that a voxel or blocky mesh primitive has a finite set of potential faces (typically 6 for a cube) and that visibility is determined by the state of adjacent voxels, the viewing direction, or external depth information.

**Adjacency and Occlusion:**
The most common mechanism is checking immediate neighbors. For a voxel at position $P$ with a face normal $N_{face}$, the neighbor position is calculated as $P_{neighbor} = P + N_{face}$. If the neighbor at $P_{neighbor}$ is solid (opaque), the face at $P$ in direction $N_{face}$ is internal and culled [S1, unofficial]. This reduces the primitive count from $6 \times N_{voxels}$ to approximately $2 \times N_{voxels}$ for dense structures.

**Camera Orientation:**
Many methods incorporate backface culling. A face is culled if its normal $\mathbf{n}$ and the view direction $\mathbf{v}$ satisfy $\mathbf{n} \cdot \mathbf{v} > 0$ (assuming $\mathbf{v}$ points from the camera to the scene) or similar conventions, indicating the face is oriented away from the viewer [S1, unofficial].

**Granularity:**
Culling can occur at different granularities:
1.  **Block Level:** Entire blocks are skipped if they are buried or outside the view frustum [S2, unofficial][S1, unofficial].
2.  **Face Level:** Individual faces of a voxel are discarded based on neighbor occupancy [S1, unofficial].
3.  **Sub-Face Level:** Partial shapes (e.g., stairs) use bitmasks to cull specific portions of a face [S1, unofficial].
4.  **Mesh Level:** Post-generation filtering removes faces based on opacity or multi-view depth consistency [S1, unofficial][S2, unofficial].

**Data Structures:**
Common structures include:
*   **Occupancy Grids:** Hash sets or dense arrays storing voxel states [S1, unofficial].
*   **Visibility Masks:** Bitmasks (e.g., 6-bit integers) tracking which faces of a voxel are visible [S1, unofficial].
*   **Vertex Packed Buffers:** Compact storage (e.g., two 32-bit integers per vertex) to minimize memory overhead during incremental updates [S1, unofficial].

## Subtopics

### [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md)
Face Coverage Mask Culling removes hidden faces of partial voxel shapes (such as stairs, slabs, or custom geometries) by representing face coverage as a bit mask. It solves the problem where a boolean "solid/empty" flag is insufficient because partial shapes have mixed visible and hidden areas on a single face. Unlike sibling methods that rely on coarse block visibility or vertex packing, this method operates at the sub-face granularity, allowing precise culling of geometry that is occluded by neighbors while preserving exposed portions of the same face.

### [ChunkMesh Vertex Packing and Management](3566-chunkmesh-vertex-packing-and-management.md)
ChunkMesh Vertex Packing and Management is a dynamic mesh generation and update technique for voxel terrain that combines face culling with compact vertex storage and incremental buffer updates. It solves the problem of high memory overhead and frequent GPU buffer uploads in voxel worlds by packing vertex attributes into two 32-bit integers and reusing vertex slots via a free-list allocator. Unlike static meshing methods, it supports incremental addition and removal of blocks, updating only the affected faces and their neighbors.

### [Voxel Backface Culling and Internal Face Masking](4569-voxel-backface-culling-and-internal-face-masking.md)
This method optimizes voxel mesh rendering by rejecting faces that are either occluded by adjacent voxels (internal faces) or oriented away from the camera (backfaces). It distinguishes itself from standard neighbor-check culling by performing visibility tests in projection space to determine front/back orientation, while using a compact bitmask to track internal occlusion, allowing for efficient geometry shader-based culling.

### [Block Visibility Culling](5334-block-visibility-culling.md)
Block Visibility Culling determines which voxel blocks require mesh updates or rendering based on the camera's position and rotation, avoiding unnecessary geometry processing when the camera has not moved significantly or changed its primary viewing direction. It operates at the coarse block level by comparing current camera coordinates against previous frame values to skip index updates, and by calculating surface orientation relative to the player to cull invisible faces. Unlike its siblings which often operate at the fine-grained face level within a block or use complex projection masks, this method relies on simple integer coordinate comparison and basic trigonometric orientation checks.

### [Extraction of Surface Points (Sobierajski)](5721-extraction-of-surface-points-sobierajski.md)
A fast display method for direct rendering of boundary surfaces by extracting only visible boundary voxels from volume data. It sets itself apart from sibling methods like [Face Culling](8735-face-culling.md) by operating in object space to generate a pre-filtered list of surface points based on viewing directions, rather than determining face visibility per voxel during mesh generation. This approach reduces the rendering load by culling back-facing voxels and projecting them to avoid holes, serving as a direct volume rendering acceleration technique.

### [Mask Projection Culling](6836-mask-projection-culling.md)
Mask Projection Culling is a visibility-based mesh filtering technique that removes faces and vertices not consistently observed across multiple camera poses by projecting 3D points into camera space and comparing their depth against rendered depth maps or binary masks. It solves the problem of retaining only geometry that contributes to the final rendered image from a specific set of viewpoints, effectively pruning occluded or background geometry. Unlike sibling methods that rely on voxel neighbor checks [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md) or simple back-face orientation [Voxel Backface Culling and Internal Face Masking](4569-voxel-backface-culling-and-internal-face-masking.md), this method uses explicit camera projection and depth validation, making it suitable for complex, non-voxelized meshes or point clouds where topological neighbor information is unavailable or expensive to compute.

### [Mesh Culling via 3D Projection and Connected Components](6963-mesh-culling-via-3d-projection-and-connected-components.md)
This method removes geometric artifacts and invisible faces from a 3D mesh by projecting its vertices into multiple camera views and comparing their depths against reference depth maps. It classifies mesh faces as "seen," "unseen," or "forecast" based on multi-view consistency, culling those that fail visibility checks or fall outside a defined bounding volume. Unlike sibling methods that rely on local voxel neighbor checks [3861-face-culling-and-visibility-optimization.md] or coverage masks [3397-face-coverage-mask-culling.md], this approach uses global camera poses to eliminate floating noise and background artifacts common in unbounded reconstructions.

### [Face Culling](8735-face-culling.md)
Face Culling is a mesh generation optimization for voxel-based volume rendering that reduces the number of vertices and triangles sent to the GPU by skipping the rendering of voxel faces that are adjacent to existing neighboring voxels. It solves the problem of naively rendering all 6 faces of every cube, including those hidden by neighbors, which results in significant overdraw and wasted rendering resources. This method sets itself apart from its siblings by focusing on immediate geometric adjacency checks during mesh construction rather than post-generation visibility tests or coverage masks.

### [Opacity-based Mesh Culling](8759-opacity-based-mesh-culling.md)
Opacity-based Mesh Culling is a post-processing optimization technique applied to polygonal meshes extracted from volumetric representations (such as Neural Radiance Fields or voxel grids) to reduce geometric complexity. It solves the problem of excessive triangle counts and low mesh utilization by discarding faces that contribute negligible visual information based on their opacity values. Unlike standard geometric face culling which relies on neighbor occupancy or backface visibility, this method operates on the material properties (opacity/density) of the extracted primitives, often resulting in a drastic reduction of the final mesh count (e.g., down to 5% of the original) while preserving visual fidelity.

### [From-Region Visibility Preprocessing](9048-from-region-visibility-preprocessing.md)
From-region visibility preprocessing is a technique that precomputes which geometric primitives are visible from a specific region of the camera space, allowing the renderer to discard occluded geometry before the frame is drawn. It solves the problem of excessive overdraw and rendering cost in large, complex scenes by shifting the visibility determination burden to an offline or semi-offline phase. Unlike sibling methods that perform per-frame culling or rely on simple axis-aligned block checks, this approach builds a spatial visibility structure (often involving occluder fusion) tailored to a defined viewing region.

### Buried Bitmask Optimization
A bitmask used to track which blocks in a chunk are 'buried' (fully surrounded by opaque blocks), allowing the renderer to skip mesh generation for those blocks [S2, unofficial]. The process involves resetting the buried bitmask for the chunk, creating mapped adjacent chunks for neighbor access, and iterating through all blocks to determine if they are buried. If all blocks in a chunk are buried, the mesh is marked as empty and generation is skipped. During mesh generation, any block that is air or has its buried flag set in the bitmask is skipped [S2, unofficial].

### Two-pass transparent meshing with opaque separation
A meshing strategy that separates opaque and transparent geometry into two passes to ensure correct rendering order and reduce overdraw, by generating opaque faces between air/transparent blocks and transparent faces only for blocks touching air [S3, unofficial]. The first pass generates opaque faces by testing for opaque blocks (ignoring transparent and air if transparent blocks exist in the chunk), ensuring opaque faces are generated between air and transparent blocks. The second pass tests for non-air blocks to mark only transparent blocks, generating faces for all transparent blocks touching air but not adding faces between transparent and solid blocks. Solid block types are filtered out during the transparent pass to prevent incorrect face addition [S3, unofficial].

### Smart Leaf Rendering
A leaf block rendering strategy that preserves alpha cutouts on outer leaf surfaces while culling internal faces to reduce vertex count [S4, unofficial]. The behavior is controlled by a global static enum `sLeafRenderMode`, which defaults to `ChunkRenderer::LeafRenderMode::Smart` [S4, unofficial].

## Background
The concept of removing interior polygons was introduced in early voxel terrain generation systems to address the performance bottleneck of rendering full cube models. In these systems, a brute-force approach would instantiate a model for every voxel, resulting in 12 triangles per cube even when only one side was visible. The optimization involved reconstructing cubes from triangles and checking for solid neighbors; if a neighbor existed, the adjacent side was not rendered [S1, unofficial]. To further optimize, meshes of all cubes were combined into a single big mesh called a 'chunk', although this introduced vertex count limits (e.g., ~65,000 vertices in Unity) [S1, unofficial]. This technique is foundational to modern voxel engines like Minecraft, where face culling is a primary optimization for rendering large worlds [S1, unofficial].

## Sources
[S1] Voxel-Based Procedural Terrain Generation | PDF | Voxel | Rendering ..., https://www.scribd.com/document/840779393/Procedural-Terrain-Generation-Using-Voxels, fetched 2026-10-29 (unofficial)
[S2] gabkhanfig/Cube, https://github.com/gabkhanfig/Cube, fetched 2026-10-27 (unofficial)
[S3] Supporting Transparent Voxel Textures, https://github.com/TanTanDev/binary_greedy_mesher_demo/issues/5, fetched 2026-10-27 (unofficial)
[S4] ldominiq/ft_minecraft, https://github.com/ldominiq/ft_minecraft, fetched 2026-10-27 (unofficial)
