# Ray Marching Optimization


## Overview
Ray Marching Optimization encompasses techniques that reduce the computational cost of evaluating participating media during ray traversal, distinct from spatial acceleration structures that skip empty space [S1, unofficial]. While [Ray-grid traversal](17-ray-grid-traversal.md) determines *where* to sample, these methods optimize the *cost* of the sampling and material evaluation at those points [S1, unofficial]. They achieve this through traversal order precomputation, memory-efficient tiling, material simplification (e.g., grayscale evaluation), conservative density gating, and geometric coherence exploitation (e.g., epipolar binary trees) [S1, unofficial][S2, S3, S4, S5][S6, S7, unofficial].

## How it works
All methods in this family build on [Ray-grid traversal](17-ray-grid-traversal.md), which advances analytically from one cell boundary to the next [S1, unofficial]. The core mechanism determines the next cell by comparing distances to upcoming boundaries on the x, y, and z axes, selecting the minimum distance to advance [S1, unofficial]. Optimization and cost models do not alter this fundamental geometric logic but instead optimize the *cost* of executing it or the *parameters* (like grid resolution) under which it runs [S1, unofficial].

## Subtopics

### [Voxel Ray Marching with Traversal Order Optimization](1251-voxel-ray-marching-with-traversal-order-optimization.md)
Renders a 3D voxel texture by advancing a ray from the camera to the back plane and compositing samples via front-to-back alpha blending. It differs from its siblings by minimizing floating-point operations through the precomputation of step deltas and determining traversal order via $t_{max}$ comparisons, rather than relying on fixed-step checks or complex acceleration structures. Choose it when geometric traversal efficiency is prioritized over memory footprint or material evaluation cost.

### [Tiled Ray Marching with Grid Sampling](2586-tiled-ray-marching-with-grid-sampling.md)
Renders a single view of a 3D volume by generating rays, sampling density via `grid_sample`, and accumulating attenuation, supporting tiling to manage memory for large images. It differs from its siblings by explicitly decoupling ray traversal from sampling, using a dense coordinate grid and GPU-accelerated interpolation instead of explicit voxel traversal or sparse structures. Choose it when memory footprint is the primary constraint and dense grid interpolation is acceptable.

### [Volume Ray Marching Optimization](14309-volume-ray-marching-optimization.md)
A set of shader-level and rendering pipeline techniques designed to reduce the computational cost of evaluating participating media, specifically for volumetric clouds. It differs from its siblings by optimizing the *cost* of density and lighting calculations at each sample point rather than skipping empty space. Choose it when material evaluation and scattering approximations are the primary bottlenecks.

### [Conservative Density Evaluation](14319-conservative-density-evaluation.md)
Uses a user-specified, inexpensive-to-evaluate "conservative density" to gate the execution of expensive material graphs (Albedo, Emissive Color, Extinction). It differs from its siblings by skipping expensive computations when the conservative density indicates no participating medium exists, rather than optimizing traversal or memory. Choose it when cloud density is sparse and material graphs are computationally heavy.

### [Epipolar Sampling with 1D Min/Max Binary Tree Optimization](16572-epipolar-sampling-with-1d-min-max-binary-tree-optimization.md)
Accelerates the inscattering integral calculation for volumetric light shafts by casting rays through an epipolar plane defined by the camera and light direction. It differs from its siblings by exploiting the geometric coherence of epipolar slices to reduce shadowing cost from linear to logarithmic per sample, rather than optimizing traversal order or grid sampling. Choose it when rendering volumetric light shafts and shadow tests are the bottleneck.

#### Indicator Grid Ray Marching
- **Definition**: A binary map cached and updated during training to signify empty areas, allowing for early termination of ray marching based on transmittance, thereby optimizing ray marching efficiency [S2].
- **Steps**: 1. Initialize the indicator grid as fully occupied, assuming all space is dense [S2]. 2. Sample a small batch of spatiotemporal points and pass them through the tiny MLP $\theta$ to obtain density [S2]. 3. Update the indicator grid based on the obtained density distribution [S2].
- **Comparison**: Compared to the original Lerplane [51] which uses a sample-net requiring extensive sampling, this method reduces unnecessary point sampling, enabling approximately 5 times faster inference (1.73 fps vs. 0.38 fps) [S2].

#### Implicit Surface Geometry Estimation via Softargmax
- **Definition**: An implicit surface representation $G$ estimated from volumetric intensity $I_{pf}$ without creating explicit meshed geometry, using ray marching and softargmax for differentiable depth estimation [S3].
- **Steps**: 1. Sample points along each ray with ray marching, and estimate intensity at each sampled point by trilinear interpolation of neighbor voxel intensities of $I_{pf}$ [S3]. 2. Estimate normal $n_g$ at vertex $x_g$ based on distances $d_N, d_S, d_E, d_W$ at neighboring ray samples in concentric hemispherical mapping. Compute normals of two triangles $\triangle d_N d_E d_S$ and $\triangle d_S d_W d_N$ via cross product, then $n_g$ is normalized sum of these normals [S3].
- **Formula**: $d_{gs} = \sum_i \omega_i d_i / \sum_i \omega_i$, where $d_i$ is ray-marched distance from $x_s$, $\omega_i = e^{\beta I_{pf,i}}$ is probability density function, $I_{pf,i}$ is volume intensity at distance $d_i$, and $\beta$ is hyperparameter [S3].
- **Parameter**: $\beta$ is a hyperparameter determining sensitivity in blending neighboring probabilities, set to 1e+3 in all experiments [S3].
- **Pitfall**: If $I_{pf}$ falls below a threshold, no surface is assumed found. Threshold is 0.05 for synthetic scenes and 0.2 for real scenes [S3].

#### Pre-computing Ray Parameters
- **Definition**: An optimization that eliminates repetitive matrix transformation computations for sampling coordinates by pre-computing ray parameters. It reduces matrix multiplications to simple arithmetic addition and subtraction [S4].
- **Layout**: Ray parameters are represented by a coordinate for the ray origin and a vector for the ray direction [S4].
- **Pitfall**: Requires a constant viewing configuration. Viewing angles and distances cannot be adjusted because pre-computation assumes rays are always fixed. A view change requires the whole dataset to be re-traversed [S4].

#### View-Dependent Visibility Optimization for Monte Carlo Volume Visualization
- **Definition**: A view-dependent optimization technique for Monte Carlo ray tracing that modifies the extinction coefficient along rays to enhance the visibility and illumination of important regions, solving the incompatibility between stepwise ray marching optimizations and free-flight distance sampling in Monte Carlo methods [S5].
- **Problem**: Existing view-dependent optimizations (like [AZD17]) rely on stepwise sampling along rays, which is fundamentally incompatible with the free-flight distance sampling used in state-of-the-art Monte Carlo methods like Woodcock tracking [S5].

#### Coarse Cone Tracing
- **Definition**: An optimisation to approximate the surface of the SDF and speed up raymarching by using cones that act as 'growing spheres' marched over multiple pixels in a pre-pass [S6, unofficial].
- **Steps**: 1. Cones are marched at a lower resolution in a pre-pass [S6, unofficial]. 2. Each pixel’s ray starts from the end of the cone rather than the camera’s position [S6, unofficial].
- **Comparison**: Reduces the number of steps each ray must make through empty space compared to standard raymarching from the camera [S6, unofficial].
- **Usage**: Used in the game Claybook [S6, unofficial].

## Operating characteristics

## Background
Ray marching optimization has evolved from basic traversal logic to sophisticated shader-level and data-structure-driven techniques [S1, unofficial]. Early optimizations focused on traversal order, such as voxel ray marching with traversal order optimization which minimizes floating-point operations by precomputing step deltas [S1, unofficial]. With the rise of GPU computing, memory management became a focus, leading to tiled ray marching with grid sampling to handle large volumes [S1, unofficial]. In real-time rendering, particularly for volumetric clouds, optimization shifted to material evaluation costs, introducing conservative density evaluation and grayscale material simplifications [S1, unofficial]. More recent approaches exploit geometric coherence, such as epipolar sampling with 1D min/max binary trees for light shafts [S1, unofficial], and training-based optimizations like indicator grids for deformable tissues [S2]. Monte Carlo methods have also seen adaptations, such as view-dependent visibility optimization to handle free-flight sampling [S5], and SDF-based optimizations like coarse cone tracing in games like Claybook [S6, unofficial]. Implicit surface geometry estimation via softargmax enables differentiable depth from volumetric intensity [S3], while pre-computing ray parameters eliminates repetitive matrix transformations for fixed viewing configurations [S4].

## Sources
[S1] dehezhang2/RISE-SDF, https://github.com/dehezhang2/RISE-SDF, fetched 2026-10-27 (unofficial)
[S2] Efficient Deformable Tissue Reconstruction via Orthogonal Neural Plane, https://doi.org/10.1109/tmi.2024.3388559, fetched 2026-10-26
[S3] Self-Calibrating, Fully Differentiable NLOS Inverse Rendering, https://doi.org/10.1145/3610548.3618140, fetched 2026-10-27
[S4] Online volume rendering of incrementally accumulated LSCEM images for superficial oral cancer detection, https://doi.org/10.5306/wjco.v2.i4.179, fetched 2026-10-30
[S5] View‐Dependent Visibility Optimization for Monte Carlo Volume Visualization, https://doi.org/10.1111/cgf.70064, fetched 2026-11-05
[S6] PDF Generating Signed Distance Fields from Triangle Meshes using Vulkan ..., https://williamwhitehouse.dev/assets/projects/2023/sdf-gen/gen-sdf-from-meshes-using-vulkan-compute-report.pdf, fetched 2026-10-30 (unofficial)
[S7] Participating media, https://imaging.cs.cmu.edu/15-468/2024_spring/lectures/lecture_12.pdf, fetched 2026-10-29 (unofficial)
