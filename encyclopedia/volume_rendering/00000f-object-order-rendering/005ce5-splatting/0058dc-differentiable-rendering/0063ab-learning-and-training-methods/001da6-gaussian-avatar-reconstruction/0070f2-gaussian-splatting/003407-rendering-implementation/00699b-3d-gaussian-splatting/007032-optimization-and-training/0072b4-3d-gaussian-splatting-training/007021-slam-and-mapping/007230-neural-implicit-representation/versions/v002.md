# Neural Implicit Representation


## Overview
Neural Implicit Representation encodes a 3D scene as a continuous function, typically a Neural Radiance Field or Signed Distance Function, rather than a discrete voxel grid or point cloud [S1]. It solves the problem of joint camera pose estimation and dense 3D reconstruction by optimizing network weights and camera poses simultaneously using photometric and geometric constraints . It differs from explicit point-based siblings like Gaussian Splatting by modeling complex geometry and appearance continuously [S1].

## Shared principles
The family relies on a neural network $F$ that maps spatial coordinates $\mathbf{x}$ to scene properties [G]. For appearance, $F$ outputs color $\mathbf{c}$ and volume density $\sigma$ (or occupancy) [S1]. For geometry, $F$ may output a Signed Distance Function (SDF) value [S1]. Rendering is performed via volume rendering or ray marching: points are sampled along camera rays, properties are predicted by $F$, and results are aggregated using alpha compositing or weighted sums [S1]. Optimization jointly updates the network weights and camera poses $T_t$ to minimize photometric ($\ell_{rgb}$) and geometric ($\ell_{depth}$ or SDF) losses [S1].

## Subtopics

### [Neural Radiance Field (NeRF) Representation](12096-neural-radiance-field-nerf-representation.md)
Encodes scenes as continuous volumetric functions or explicit Gaussian primitives. It supports both implicit neural field evaluation and explicit GPU-friendly Gaussian splatting, trading memory for speed while maintaining competitive training times.

### [Neural Implicit SLAM](22159-neural-implicit-slam.md)
Represents the scene as a continuous neural implicit function (NeRF or SDF) for joint localization and mapping. It differs from Gaussian Splatting by using continuous volumetric representation requiring ray marching, whereas Gaussian Splatting uses explicit, differentiable 3D Gaussians for object-order rendering.

### PIN-SLAM
A globally consistent SLAM system using a point-based implicit neural (PIN) representation [S1]. It replaces grid structures with neural feature points, offering spatial flexibility and elastic correction during loop closure [S1]. Mapping alternates with odometry; the mapping step uses a hybrid neural SDF representation with explicit neural points and implicit MLP decoders [S1]. Odometry is performed correspondence-free via second-order scan-to-map optimization [S1]. A sliding window replay buffer ensures stable incremental updates [S1]. Loop closures trigger pose graph optimization and elastic deformation of neural points, enabling consistent large-scale mapping [S1].

## Background
The approach originated in continuous scene representation literature and was adapted for simultaneous localization and mapping (SLAM) to enable dense, consistent mapping without explicit mesh extraction [S1]. Recent developments include PIN-SLAM (January 2024), which uses a point-based implicit neural representation with elastic loop closure correction [S1], KN-SLAM (March 2024), which integrates local feature correspondences for coarse pose initialization and joint photometric-feature loss optimization [S1], and SLAIM (April 2024), which introduces a coarse-to-fine tracking pipeline with Gaussian-filtered image signals for improved photometric bundle adjustment [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
