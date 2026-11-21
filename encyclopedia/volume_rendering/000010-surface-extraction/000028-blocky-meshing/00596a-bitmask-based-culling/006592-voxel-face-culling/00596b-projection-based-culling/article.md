# Projection-based Culling


## Overview
Projection-based culling is a family of optimization techniques that determine the visibility of geometric primitives by transforming their vertices into camera or projection space and testing them against depth buffers, masks, or orientation vectors. It solves the problem of rendering or processing large sets of geometry (voxels or meshes) by discarding primitives that are occluded, back-facing, or outside the viewing frustum, thereby reducing the workload for the rasterizer or subsequent processing stages. This family is distinguished from spatial partitioning methods (like [Block Visibility Culling](5334-block-visibility-culling.md)) by relying on explicit per-vertex projection tests rather than hierarchical spatial queries, and from simple face culling ([Face Culling](13637-face-culling.md)) by incorporating view-dependent visibility checks.

## Shared principles
All methods in this family share the core mechanism of projecting 3D coordinates into a 2D view space to evaluate visibility. The common mathematical steps are:

1.  **Coordinate Transformation**: Vertices $v$ in world space are transformed to camera space using the world-to-camera matrix $W$:
    $$ v_{cam} = W \cdot v_{world} $$
    For voxel methods, this may involve transforming corner positions to view space using the inverse projection diagonal to approximate direction .

2.  **Projection**: Camera-space coordinates $(x_c, y_c, z_c)$ are projected to 2D screen coordinates $(u, v)$ using intrinsic parameters $K = \text{diag}(f_x, f_y, 1)$ and principal point $(c_x, c_y)$:
    $$ u = f_x \frac{x_c}{z_c} + c_x $$
    $$ v = f_y \frac{y_c}{z_c} + c_y $$
    

3.  **Visibility Testing**: A primitive is considered visible if it passes one or more of the following tests:
    *   **Frustum Test**: The projected coordinates lie within the image bounds $[0, W) \times [0, H)$ and the depth $z_c > 0$ .
    *   **Depth Test**: The vertex depth $z_c$ is less than the sampled depth $d_{sample}$ from a reference depth map within a tolerance $\epsilon$:
        $$ z_c < d_{sample} + \epsilon $$
        
    *   **Mask Test**: The projected point falls within a binary mask $M$ indicating valid regions .
    *   **Backface Test**: For voxels, the dot product of the view direction and the transformed corner sum determines front/back orientation .

4.  **Aggregation**: For mesh methods, visibility counts are aggregated across multiple views. A vertex is retained if it is visible in at least $N$ views .

## Subtopics

### [Voxel Backface Culling and Internal Face Masking](4569-voxel-backface-culling-and-internal-face-masking.md)
This method applies projection-based culling to voxel grids. It determines the visibility of voxel faces by testing the orientation of the voxel's diagonal corners in view space. It differs from mesh methods by operating on implicit voxel geometry and using bitmask flags to prevent double-rendering of shared internal faces. It is chosen when rendering large voxel datasets where internal faces must be suppressed to reduce overdraw.

### [Mask Projection Culling](6836-mask-projection-culling.md)
This technique filters explicit triangular meshes by projecting vertices into multiple camera views and checking them against binary segmentation masks or rendered depth maps. It differs from other projection methods by relying on per-pixel mask sampling (often with dilation) rather than just depth comparison. It is chosen when high-precision visibility is required based on semantic masks or when depth maps are unreliable.

### [Mesh Culling via 3D Projection and Connected Components](6963-mesh-culling-via-3d-projection-and-connected-components.md)
This method removes geometric artifacts from meshes (e.g., from Gaussian splatting) by validating vertex visibility against reference depth maps across multiple views. It distinguishes itself by classifying points as "seen," "unseen," or "forecast" and optionally using connected component analysis to remove small floating artifacts. It is chosen for cleaning up noisy 3D reconstructions where multi-view consistency is key.

## Background
Projection-based culling is a fundamental technique in computer graphics, originating from standard rasterization pipelines where backface culling and frustum culling are performed per primitive. The application of multi-view depth projection for mesh cleaning and voxel face culling has been developed in the context of large-scale terrain rendering and 3D reconstruction from multi-view stereo data. The specific implementations described here are derived from unofficial notes on voxel rendering optimizations and 3D Gaussian splatting post-processing pipelines .

## Sources
