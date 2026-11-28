# Geometric Structure Constraint

Also called: Normal Smoothness and Bilateral Edge-Aware Constraint

## Overview
Geometric Structure Constraint is a method that integrates explicit geometric priors directly into the NeRF optimization loop to guide density field learning and improve elevation estimates [S1]. Unlike post-processing refinement, it unifies physical imaging and geometric constraints so that radiometric decomposition participates in optimization, enabling more accurate 3D reconstruction under complex satellite imaging conditions [S1]. It sets itself apart from siblings by explicitly combining normal smoothness with a bilateral edge-aware constraint to preserve sharp geometric boundaries while smoothing noise [S1].

## Shared principles
All methods in this family build on [Specific Model Architectures](8585-specific-model-architectures.md) but diverge in how the geometric information is integrated into the optimization loop. The shared mechanism involves augmenting the standard photometric loss with geometric terms or modifying the ray sampling process to respect geometric boundaries.

**Geometric Loss Functions:**
Many methods introduce explicit loss terms that penalize deviations from geometric priors. Common forms include:
1.  **Depth Consistency Loss:** Minimizes the difference between the rendered depth (computed as the expected depth along the ray) and a prior depth map $D_{prior}$:
    $$ L_{depth} = \| D_{rendered} - D_{prior} \|_1 $$
2.  **Normal Smoothness:** Encourages surface normals derived from the density field to be smooth or aligned with estimated normals [S1].
3.  **Epipolar Constraints:** Enforces that 3D points reconstructed from matched features in different views lie on the corresponding epipolar lines [S1].

**Depth-Guided Sampling:**
Instead of uniform or hierarchical sampling along the entire ray, these methods use depth priors to concentrate samples near the estimated surface. This reduces computational cost and mitigates the "transparent" artifacts common in standard NeRF when geometry is ambiguous. For example, samples are clustered around the depth value predicted by a monocular depth estimator [S1].

**Signed Distance Fields (SDF):**
Some variants replace the volumetric density $\sigma$ with a Signed Distance Field (SDF). The rendering integral is modified to integrate weights based on the SDF value relative to a truncation distance $tr$, effectively creating a thin shell around the surface rather than a volumetric blob. The weight for a sample at distance $d$ is often computed as:
$$ w = \sigma\left(\frac{d}{tr}\right) \cdot \sigma\left(-\frac{d}{tr}\right) $$
This allows for direct mesh extraction via Marching Cubes [S1].

## Subtopics

### [SfMNeRF](8664-sfmnerf.md)
Uses SfM priors for initialization and regularization.

### [DASNeRF](8668-dasnerf.md)
Integrates depth and surface normal supervision.

### [GeoNeRF](8710-geonerf.md)
Focuses on geometric consistency across views.

### [Neural RGB-D Surface Reconstruction](11352-neural-rgb-d-surface-reconstruction.md)
Combines RGB-D data for surface reconstruction.

### [DINER](11354-diner.md)
Uses depth inpainting for novel view synthesis.

### [Volume Rendering Prior (VRPrior)](14662-volume-rendering-prior-vrprior.md)
Applies volume rendering priors for regularization.

## Background
The Geometric Structure Constraint was proposed in the context of DSM (Digital Surface Model) reconstruction from multi-view satellite images [S1]. The method addresses the difficulty of achieving optimal reconstruction results when geometric refinement is applied only as a post-processing step, as illumination and geometry are refined only after the radiance field has been estimated [S1]. By decomposing illumination, albedo, and surface normals within the NeRF framework, and modeling sunlight and ambient illumination separately, the method prevents shadow-induced brightness variation from being absorbed into albedo [S1]. The geometric branch consists of a normal-smoothing branch and a bilateral edge-aware branch, which play complementary roles in guiding the network to accurately learn the density field distribution [S1].

## Sources
[S1] Shading and Geometric Constraint Neural Radiance Field for DSM Reconstruction from Multi-View Satellite Images, https://doi.org/10.3390/rs18071091, fetched 2026-11-05
