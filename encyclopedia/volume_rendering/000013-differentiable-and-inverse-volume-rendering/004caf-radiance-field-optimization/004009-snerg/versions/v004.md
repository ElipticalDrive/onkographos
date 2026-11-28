# SNeRG
Also called: Deferred Shading NeRF, SNeRG (Deferred Shading NeRF), Sparse Neural Radiance Fields with View-dependent Effects

## Overview
SNeRG reduces the computational cost of Neural Radiance Fields (NeRF) by decomposing the radiance field into a deferred shading model, limiting Multilayer Perceptron (MLP) evaluations to one per ray [S1]. Unlike standard NeRF which queries view-dependent color for every sample, SNeRG caches diffuse colors and uses a small network for view-dependent effects, enabling real-time rendering [S2, unofficial].

## How it works
**Inputs:** Camera poses, intrinsics, and training images [G].
**Outputs:** Rendered images and an optimized scene representation [G].

**Data Structures:**
1. **Sparse 3D Grid:** Stores density ($\sigma$) and diffuse color ($c_d$) at voxel centers, using a hash map or sparse array for memory efficiency [S1].
2. **3D Texture Atlas:** Caches baked diffuse colors for fast lookup [S2, unofficial].
3. **Feature Vectors ($f$):** Stored per voxel or interpolated, used to modulate view-dependent effects [S1].
4. **Small Network:** A lightweight MLP that takes spatial and directional inputs to output view-dependent color [S2, unofficial].

**Steps:**
1. **Ray Marching:** For each pixel, cast a ray through the scene and use the sparse 3D grid to quickly locate surface boundaries and skip empty space [S2, unofficial].
2. **Sample Query:** For each sample point $x$ along the ray, query its eight neighbors on a low-resolution 3D grid and project $x$ onto three axis-aligned planes to query four neighbors on each high-resolution 2D grid [S1]. Trilinearly interpolate density $\sigma$, diffuse color $c_d$, and feature vectors $f$ from these neighbors [S1].
3. **Deferred Shading:** Compute transmittance $T$ and accumulated density using standard volume rendering integration [G]. The base color is the interpolated diffuse color $c_d$ [S1]. Pass position $x$, features $f$, and view direction $v$ to the small network to compute the final view-dependent color $c$ [G].
4. **Rendering:** Composite the final pixel color using the accumulated transmittance and the view-dependent color $c$ [G].
5. **Optimization:** Minimize the difference between rendered and ground truth images [G]. Apply a sparsity loss to reduce the VRAM usage of sparse voxels [S2, unofficial].

**Formulas:**
The radiance field is decomposed into density $\sigma$, diffuse color $c_d$, and features $f$ [S1].
The final color $C(r)$ for a ray $r$ is:
$$ C(r) = \sum_{i=1}^{N} T_i (1 - e^{-\sigma_i \delta_i}) c_i $$
where $T_i = e^{-\sum_{j=1}^{i-1} \sigma_j \delta_j}$, $\delta_i = x_{i+1} - x_i$ is the distance between adjacent samples, and $c_i$ is the view-dependent color computed by the small network [S1].

**Parameters:**
* Grid resolution: Low-resolution 3D grid for density/color, high-resolution 2D projections for spatial queries [S1].
* Network size: Small MLP for view-dependent effects [S2, unofficial].

**Edge Cases:**
* Sparse grid boundaries require careful handling to avoid artifacts [S2, unofficial].

## Subtopics

### [Hierarchical Grid Optimization](1414-hierarchical-grid-optimization.md)
Uses hierarchical structures for adaptive resolution; chosen when scene complexity varies significantly across space.

### [3D Gaussian Splatting](1795-3d-gaussian-splatting.md)
Uses explicit Gaussians instead of implicit fields; chosen for rasterization-based real-time rendering.

### [Neural Transient Fields (NeTF)](4760-neural-transient-fields-netf.md)
Handles dynamic scenes; chosen when temporal coherence and motion are required.

### [Neural Radiance Fields](4972-neural-radiance-fields.md)
The original method SNeRG optimizes; chosen when maximum offline fidelity is prioritized over speed.

### [NeRF++](6198-nerf.md)
Handles unbounded scenes; chosen for outdoor or large-scale environments.

### [Generalized Rendering Models](6681-generalized-rendering-models.md)
Extends to general light transport; chosen for complex material and lighting interactions.

### [Optimization Procedures](7150-optimization-procedures.md)
Training strategies; chosen to improve convergence and stability.

### [Simple-RF](7279-simple-rf.md)
Simplified radiance field representations; chosen for lightweight deployment.

### [VDN-NeRF](7383-vdn-nerf.md)
View-dependent networks; chosen when explicit view modulation is preferred.

### [DeRF (Decomposed Radiance Fields)](7673-derf-decomposed-radiance-fields.md)
Decomposes fields for efficiency; chosen for memory-constrained pipelines.

### [Dual Rendering Equation Optimization](8167-dual-rendering-equation-optimization.md)
Optimizes both forward and inverse problems; chosen for inverse graphics tasks.

### [Planar Reflection-Aware NeRF](8312-planar-reflection-aware-nerf.md)
Handles reflections; chosen for scenes with glossy or mirror surfaces.

### [Tetra-NeRF](8932-tetra-nerf.md)
Uses tetrahedral meshes; chosen for structured domain decomposition.

### [Light Field and Lumigraph Methods](9271-light-field-and-lumigraph-methods.md)
Sampling-based approaches; chosen for dense camera array inputs.

### [Generalizable Radiance Fields](9480-generalizable-radiance-fields.md)
Zero-shot generalization; chosen for few-shot or unseen scene synthesis.

### [General Radiance Field (GRF)](9638-general-radiance-field-grf.md)
Unified representation; chosen for multi-modal scene modeling.

### [Closed-Form Color Field Estimation](10574-closed-form-color-field-estimation.md)
Analytical solutions; chosen for fast, non-iterative color prediction.

### [Surface-SDF Representation](10778-surface-sdf-representation.md)
Uses signed distance functions; chosen for sharp surface reconstruction.

### [Neural Illumination Fields (NeIF)](10930-neural-illumination-fields-neif.md)
Models lighting separately; chosen for relighting and material editing.

### [NeRF-XL Joint Training](10947-nerf-xl-joint-training.md)
Large-scale training; chosen for city-scale or massive datasets.

### [Physical and Material Priors](11198-physical-and-material-priors.md)
Incorporates physical properties; chosen for physically plausible synthesis.

### [LiDAR-NeRF](11289-lidar-nerf.md)
Uses LiDAR data; chosen for metrically accurate reconstruction.

### [Deformable and Dynamic Fields](11311-deformable-and-dynamic-fields.md)
Handles motion; chosen for non-rigid scene capture.

### [HDR-NeRF](11334-hdr-nerf.md)
High dynamic range; chosen for scenes with extreme lighting variations.

### [GNeuVox](11357-gneuvox.md)
GPU-optimized voxels; chosen for high-throughput voxel rendering.

### [NeRFifyMesh](11365-nerfifymesh.md)
Combines meshes and NeRF; chosen for hybrid geometry-texture pipelines.

### [Progressively Deblurring Radiance Field (PDRF)](11382-progressively-deblurring-radiance-field-pdrf.md)
Handles motion blur; chosen for dynamic capture with camera shake.

### [Geometry Pseudo Label via Image Warping](11450-geometry-pseudo-label-via-image-warping.md)
Uses warping for geometry; chosen for self-supervised depth estimation.

### [Canonical Neural Radiance Field for X-ray Tomography](11451-canonical-neural-radiance-field-for-x-ray-tomography.md)
X-ray specific; chosen for medical or industrial volumetric imaging.

### [Tensorial and Low-Rank Representations](11617-tensorial-and-low-rank-representations.md)
Compresses fields; chosen for storage-constrained applications.

### [Hybrid Explicit-Implicit Representations](12240-hybrid-explicit-implicit-representations.md)
Combines explicit and implicit; chosen for balancing speed and detail.

### [Explicit Voxel/Grid Representations](12527-explicit-voxel-grid-representations.md)
Grid-based methods; chosen for deterministic memory allocation.

### [Neural Residual Field (ReRF)](12546-neural-residual-field-rerf.md)
Models residuals; chosen for incremental scene updates.

### [Neural 3D Volume Decomposition](12553-neural-3d-volume-decomposition.md)
Decomposes volumes; chosen for modular scene editing.

### [Shading-Guided NeRF Synthesis](12837-shading-guided-nerf-synthesis.md)
Uses shading for guidance; chosen for improved surface normal accuracy.

## Operating characteristics
* Time complexity is reduced compared to standard NeRF due to fewer MLP evaluations [S1].
* Memory complexity is reduced via sparse voxels and caching [S2, unofficial].
* Enables real-time view synthesis [S2, unofficial].
* Strengths include fast rendering and memory efficiency [S1][S2, unofficial].
* When to use: real-time applications and large scenes [S2, unofficial].
* When not to use: high-fidelity offline rendering where maximum detail is required [G].

## Background
SNeRG was introduced by Hedman et al. in "Baking Neural Radiance Fields for Real-Time View Synthesis" (ICCV 2021) [S1]. It builds on the original NeRF work which parameterized a radiance field using an MLP [S1]. The method addresses the high computational cost of NeRF by decomposing the radiance field and using deferred shading [S1].

## Sources
[S1] MERF: Memory-Efficient Radiance Fields for Real-time View Synthesis in Unbounded Scenes, https://doi.org/10.1145/3592426, fetched 2026-10-26
[S2] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
