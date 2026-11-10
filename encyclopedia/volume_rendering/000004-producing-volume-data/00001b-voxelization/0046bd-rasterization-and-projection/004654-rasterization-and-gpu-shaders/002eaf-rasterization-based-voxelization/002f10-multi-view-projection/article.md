# Multi-view projection

Also called: Multi-view rasterization, Orthographic voxelization, 3-pass voxelization

## Overview
Multi-view projection is a family of rasterization-based voxelization techniques that convert triangle meshes or depth maps into a 3D volumetric representation by rendering scene geometry from multiple orthogonal viewpoints [S1, S2, unofficial]. It solves the problem of geometric gaps and occlusion artifacts inherent in single-view projection by ensuring every surface element is visible from at least one cardinal direction (X, Y, Z) [S1, S2, unofficial]. It differs from [Single 2D Projection Voxelization](3482-single-2d-projection-voxelization.md) by using three views to cover all surface normals rather than a single dominant axis, and from [3D Grid Rasterization](903-3d-grid-rasterization.md) by relying on standard GPU rasterization pipelines rather than explicit 3D grid traversal [S2, unofficial].

## Shared principles
All methods in this family share the core mechanism of projecting 3D geometry onto 2D planes aligned with the principal axes of the scene's bounding box, then aggregating these projections into a 3D structure.

1.  **Orthographic Projection:** Cameras are positioned at infinity along the X, Y, and Z axes. The projection plane is perpendicular to the viewing direction. For example, a view along the Z-axis projects onto the XY plane [S1, S2, unofficial].
2.  **Bounding Box Alignment:** The scene is enclosed in an axis-aligned bounding box (AABB) defined by $bb_{min}$ and $bb_{max}$. The voxel grid resolution $R = (R_x, R_y, R_z)$ maps directly to this box [S1, S2, unofficial].
3.  **Atomic Aggregation:** When multiple fragments or depth values map to the same voxel coordinate, atomic operations or specific blending rules are used to resolve conflicts, ensuring occupancy is marked if *any* view intersects the voxel [Parent, G].
4.  **Coordinate Transformation:** A transformation matrix maps clip-space fragment coordinates to discrete voxel indices $(i, j, k)$. This typically involves scaling by the resolution and translating by the bounding box minimum [S1, S2, unofficial].

## Subtopics

### [Single 2D Projection Voxelization](3482-single-2d-projection-voxelization.md)
This method produces a 6-separating (tunnel-free) voxelization using only a single 2D rasterization pass per triangle, determined by the dominant axis of the triangle's normal vector. It leverages the "thinness" property to prove that projecting along the dominant axis is sufficient to capture all voxels in the corresponding column, eliminating the need for three orthogonal passes. It is chosen when computational cost must be minimized by reducing rasterization overhead, accepting the complexity of edge-case handling for pixels intersected only by triangle boundaries.

### [Orthographic Multi-View Voxelization](3892-orthographic-multi-view-voxelization.md)
Orthographic Multi-View Voxelization rasterizes scene geometry into a 3D texture or a compressed 2D "worksheet" representation using three orthographic cameras aligned with the principal axes of the scene's bounding box. It solves the problem of generating a volumetric representation for subsequent volume rendering (such as cone tracing) by ensuring every surface element is visible from at least one viewpoint, thereby preventing "shadow" voxels caused by occlusion or grazing angles. It differs from [Single 2D Projection Voxelization](3482-single-2d-projection-voxelization.md) by using three views to cover all surface normals, and from [3-Pass Voxelization](5300-3-pass-voxelization.md) by typically performing the synthesis in a single GPU-driven pass or via efficient 2D-to-3D transcoding rather than explicit CPU-side merging of three separate 3D volumes.

### [3-Pass Voxelization](5300-3-pass-voxelization.md)
3-Pass Voxelization is a rasterization-based technique that eliminates cracks and missing data in the voxel grid by rendering the scene geometry from three orthogonal view directions aligned with the principal axes (X, Y, and Z). Unlike single-view projection methods which suffer from geometric distortion or gaps when viewing angles are oblique, this method ensures accurate voxel occupancy by covering the volume from every cardinal direction. It is chosen when robustness against occlusion artifacts is prioritized over the computational cost of multiple draw calls.

### [Volumetric Representation via Depth Map Registration](6435-volumetric-representation-via-depth-map-registration.md)
This method constructs a 3D volumetric representation from a sparse set of calibrated multi-view images by registering estimated depth maps into a voxel grid. It specifically addresses non-Lambertian reflections by using individual minimum cost per view rather than a global threshold, and represents scene geometry as a sparse voxel cloud where voxel values indicate the number of views mapping to that location. Unlike static voxelization methods that simply mark occupancy, this approach includes an iterative refinement stage to smooth sharp corners and remove scattering voxels, ensuring the volume accurately represents the real scene for virtual view synthesis. It builds on [Multi-view projection](5937-multi-view-projection.md) by utilizing the registered depth maps as the primary geometric input.

### Multi-View Stereo Cost Volume Construction
#### definition
- A cost volume $V_c$ is constructed from feature volumes $\{Vf_i\}$ of $N$ reference views to perceive coarse scene structure and provide scene priors for sampling, skipping blank areas [S1].
#### parameter
- $L$: Number of depth layers in the cost volume. Typical value: 16 [S1].
#### pitfall
- Checkerboard artifacts caused by deconvolution layers in the Res-Unet processing the cost volume are eliminated by replacing deconvolution with combinations of upsampling and convolution layers [S1].

## Background
The concept of multi-view projection for voxelization originates from the need to robustly convert triangle meshes into voxel grids for volume rendering and collision detection. Early implementations relied on 3-pass rasterization to ensure no gaps existed between voxels [S1, S2, unofficial]. The technique has been widely adopted in real-time graphics for voxel cone tracing [S1] and in multi-view stereo (MVS) for 3D reconstruction from images [S1]. Recent developments, such as NeRF-SDP, utilize multi-view stereo matching to construct cost volumes for neural radiance fields, extending the traditional rasterization approach to deep learning pipelines [S1].

## Sources
[S1] NeRF-SDP: Efficient Generalizable Neural Radiance Field with Scene Depth Perception, https://doi.org/10.1145/3595916.3626380, fetched 2026-10-30
