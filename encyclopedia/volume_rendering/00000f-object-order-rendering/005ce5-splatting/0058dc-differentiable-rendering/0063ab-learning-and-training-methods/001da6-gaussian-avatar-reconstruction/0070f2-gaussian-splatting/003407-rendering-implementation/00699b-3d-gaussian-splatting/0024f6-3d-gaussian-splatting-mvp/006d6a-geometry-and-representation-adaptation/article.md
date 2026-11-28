# Geometry and Representation Adaptation

Also called: Geometry-aware 3DGS, Surface-constrained Gaussian Splatting

## Overview
This family of methods adapts the volumetric 3D Gaussian Splatting representation to better handle geometric constraints, sparse inputs, or surface-oriented rendering tasks. While standard 3DGS relies on dense multi-view photometric consistency to implicitly define geometry, these variants introduce explicit geometric supervision, surface parameterizations, or thickness regularization to resolve ambiguities inherent in sparse or deformable scenarios. They differ from standard 3DGS by decoupling or constraining the spatial distribution of Gaussians to align with known geometric priors (meshes, depth maps, or height fields) rather than relying solely on appearance optimization.

## Shared principles
All methods in this family retain the core differentiable rasterization pipeline of 3D Gaussian Splatting [3410-3d-gaussian-splatting-mvp.md] but modify the optimization objective or data representation to incorporate geometric information.

1.  **Geometric Supervision:** Methods introduce loss terms that enforce consistency between rendered outputs and external geometric signals (depth maps, mesh surfaces, or TSDF fields). This is often combined with standard photometric losses ($\mathcal{L}_{color}$).
2.  **Representation Modification:** The standard set of independent 3D Gaussians $\{G_i\}$ is either:
    *   Constrained to lie on a surface (Height-Field).
    *   Projected onto a 2D texture atlas (UV Parameterization).
    *   Dynamically expanded from sparse seeds (Unpooling).
3.  **Differentiable Geometry:** All geometric constraints are differentiable with respect to Gaussian parameters to allow gradient-based optimization.

## Subtopics

### [3D Gaussian Splatting with Gaussian Unpooling](6133-3d-gaussian-splatting-with-gaussian-unpooling.md)
*   **What it is:** A method for Few-Shot Gaussian Splatting (FSGS) that reconstructs scenes from sparse views ($N < 10$).
*   **Distinction:** Unlike dense-view baselines, it initializes Gaussians from Structure-from-Motion (SfM) points and uses "Gaussian unpooling" to expand the representation. It incorporates external geometric supervision via monocular depth estimation to resolve ambiguity.
*   **When to choose:** Use when input views are extremely sparse and standard 3DGS fails due to under-constrained geometry.

### [3D Gaussian Splatting with Mesh UV Parameterization](8046-3d-gaussian-splatting-with-mesh-uv-parameterization.md)
*   **What it is:** A method for real-time animated human reconstruction that projects Gaussian attributes into a 2D UV texture space.
*   **Distinction:** Decouples spatial appearance from 3D coordinates, enabling efficient pose-dependent deformation via mesh topology. It differs from volumetric methods by storing attributes in a 2D atlas.
*   **When to choose:** Use for animatable human avatars where pose-dependent deformation and detailed clothing capture are required.

### [H-2DGS (Height-Field 2D Gaussian Splatting)](21287-h-2dgs-height-field-2d-gaussian-splatting.md)
*   **What it is:** A method that converts volumetric 3DGS into a thin, deformable cortical surface using curved height-field Gaussians initialized on a TSDF zero level set.
*   **Distinction:** Solves depth ambiguity and volumetric thickness issues (common in medical imaging) by enforcing a surface-oriented representation. It uses a three-stage pipeline: photometric bootstrap, ray-wise density concentration, and TSDF-guided conversion.
*   **When to choose:** Use for medical imaging or scenarios with limited view coverage where a sharp, thin surface is preferred over a volumetric blob.

## Background
3D Gaussian Splatting [3410-3d-gaussian-splatting-mvp.md] was originally designed for dense multi-view reconstruction. However, applications such as few-shot learning, animated avatars, and medical imaging revealed limitations in handling sparse data, deformation, and surface sharpness. These adaptations emerged to extend 3DGS capabilities to these specific domains by integrating geometric priors.

