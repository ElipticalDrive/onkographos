# Multi-view projection
Also called: Multi-view projection voxelization, Multi-view rasterization

## Overview
Multi-view projection voxelization generates a volumetric representation of a triangle mesh by rasterizing the geometry from multiple camera viewpoints, typically aligned with the principal axes of the scene's bounding box. It solves the problem of "shadow" voxels and geometric distortion inherent in single-view projections by ensuring every surface element is visible from at least one viewpoint, thereby preventing occlusion artifacts and ensuring 6-connectivity (tunnel-free) coverage [S1, S2, unofficial]. It differs from [Single 2D Projection Voxelization](3482-single-2d-projection-voxelization.md) by using multiple views to cover all surface normals, and from [3D Grid Rasterization](903-3d-grid-rasterization.md) by relying on standard GPU rasterization pipelines rather than explicit 3D grid traversal.

## Shared principles
All methods in this family share the core mechanism of projecting 3D geometry onto 2D planes and aggregating the resulting fragments into a 3D voxel grid.

1.  **Camera Configuration:** Orthographic cameras are positioned along specific axes (e.g., X, Y, Z) looking towards the scene's axis-aligned bounding box (AABB). The view frustum is expanded slightly (e.g., by a factor of 1.02 or a fixed offset like 0.2 units) to ensure complete coverage of the bounding box corners [S1].
2.  **Rasterization:** Triangles are rasterized into 2D buffers (sheet buffers) or directly into a 3D texture. Depth testing is typically disabled or configured to allow write-through to capture all intersecting fragments, as the goal is volumetric coverage rather than surface visibility [S1, S2, unofficial].
3.  **Aggregation:** Fragments from different views are merged into the final voxel grid. This is often an atomic "OR" operation for occupancy or a blending operation for attributes. The aggregation ensures that if a voxel is hit by any view, it is marked as occupied [Parent, G].
4.  **Coordinate Mapping:** Clip-space coordinates are transformed into voxel indices using a scale and offset derived from the scene's bounding box and the target resolution $R = (R_x, R_y, R_z)$:
    $$ i = \text{floor}\left( \frac{p_{clip}.xyz - bb_{min}}{bb_{max} - bb_{min}} \times R \right) $$
    [S1, S2, unofficial].

## Subtopics

### [Single 2D Projection Voxelization](3482-single-2d-projection-voxelization.md)
Produces 6-separating voxelization using only one 2D projection along the dominant axis of a triangle's normal vector. It reduces computational cost by dynamically determining the projection direction based on geometry orientation, leveraging the thinness property of the resulting voxel set. It is chosen when performance is critical and the geometry is planar or thin.

### [Orthographic Multi-View Voxelization](3892-orthographic-multi-view-voxelization.md)
Rasters scene geometry into a 3D texture or a compressed 2D "worksheet" using three orthographic cameras aligned with the principal axes. It ensures every surface element is visible from at least one viewpoint. It differs from Single 2D Projection by using three views to cover all normals, and from 3-Pass Voxelization by typically performing synthesis in a single GPU-driven pass or via efficient 2D-to-3D transcoding.

### [3-Pass Voxelization](5300-3-pass-voxelization.md)
Eliminates cracks and missing data by projecting geometry from three orthogonal directions (X, Y, Z). It ensures complete volumetric coverage by aggregating fragments from all three principal axes, addressing geometric distortion inherent in 2D projections. It is chosen when strict topological correctness (no tunnels) is required for complex, non-planar meshes.

### [Volumetric Representation via Depth Map Registration](6435-volumetric-representation-via-depth-map-registration.md)
Constructs a 3D volumetric representation from a sparse set of calibrated multi-view images by registering estimated depth maps into a voxel grid. It addresses non-Lambertian scenes using individual minimum cost per view and represents geometry as a sparse voxel cloud. It distinguishes itself by incorporating an iterative refinement loop that removes isolated voxels and smooths sharp corners.

### Multi-View Stereo Cost Volume Construction
**Definition:** A cost volume $V_c$ is constructed from feature volumes $\{Vf_i\}$ of $N$ reference views to perceive coarse scene structure and provide scene priors for sampling, skipping blank areas [S1].
**Parameter:** $L$: Number of depth layers in the cost volume. Typical value: 16 [S1].
**Pitfall:** Checkerboard artifacts caused by deconvolution layers in the Res-Unet processing the cost volume are eliminated by replacing deconvolution with combinations of upsampling and convolution layers [S1].

## Background
Multi-view projection voxelization is a foundational technique in computer graphics for converting triangle meshes into voxel grids for volume rendering, ray marching, and neural radiance fields (NeRF). The approach leverages the efficiency of GPU rasterization pipelines, which are highly optimized for 2D triangle projection, to bypass the computational complexity of explicit 3D intersection tests. The use of multiple views (typically three orthogonal axes) was established to solve the "shadow" problem where single-view projections fail to capture surfaces facing away from the camera or those at grazing angles. The technique has evolved to include efficient memory layouts like 2D worksheets for 3D data compression and iterative refinement for multi-view stereo consistency.

## Sources
[S1] NeRF-SDP: Efficient Generalizable Neural Radiance Field with Scene Depth Perception, https://doi.org/10.1145/3595916.3626380, fetched 2026-10-30
