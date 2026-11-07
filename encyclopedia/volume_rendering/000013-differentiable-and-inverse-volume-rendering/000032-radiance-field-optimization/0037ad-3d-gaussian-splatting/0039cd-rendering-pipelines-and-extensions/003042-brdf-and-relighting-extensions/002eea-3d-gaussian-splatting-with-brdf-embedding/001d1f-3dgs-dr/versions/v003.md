# 3DGS-DR
## Overview
3DGS-DR is a method that incorporates classical reflection and shading concepts into the 3D Gaussian Splatting framework [S1, unofficial]. It addresses the limitation of standard 3DGS, which uses an emission-absorption model that bakes lighting conditions into spherical harmonics coefficients, thereby limiting performance on highly reflective surfaces and preventing relighting [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Multi-view images of a scene.
*   **Outputs:** A set of 3D Gaussians augmented with shading parameters [S1, unofficial]. Exact output layouts are not given in the sources [S1, unofficial].

**Data Structures:**
Each Gaussian stores standard 3DGS attributes (position, covariance, opacity) plus parameters for classical reflection and shading [S1, unofficial]. Exact field layouts, sizes, and bit-packing are not given in the sources [S1, unofficial].

**Numbered Steps:**
The training and optimization steps are not given in the sources [S1, unofficial].

**Formulas:**
Pixel color is computed by volume rendering the Gaussians along the ray, where the primitive color term is derived from a classical shading model [S1, unofficial]. The exact BRDF formula and shading computation steps are not given in the sources [S1, unofficial].

**Parameters:**
*   BRDF model type: Not given in the sources [S1, unofficial].
*   Learning rate: Not given in the sources [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Aliasing:** Standard 3DGS uses a dilation filter which fails to suppress high-frequency artifacts when varying focal length or camera distance [S1, unofficial]. The specific impact or mitigation for 3DGS-DR is not given in the sources [S1, unofficial].

## Subtopics

### iVR-GS Training Setup
A training configuration for iVR-GS that uses a Blinn-Phong shading model to decompose each Gaussian's color into intrinsic color and lighting components, enabling post-training appearance and illumination editing [S2, unofficial]. It composes multiple base models, each trained under one basic transformation function (TF) [S2, unofficial]. For lighting-free evaluation, training images are generated from DVR renders with lighting disabled to ensure identical rendering settings [S2, unofficial]. Minor brightness variations may persist in rendered outputs despite disabling lighting, due to the inherent view-dependent nature of the iVR-GS training formulation [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1, unofficial].
*   **Memory Complexity:** Not given in the sources [S1, unofficial].
*   **Strengths:** Enables classical reflection and shading, addressing the emission-absorption model's limitation on highly reflective surfaces and enabling relighting [S1, unofficial].
*   **Limitations:** Not given in the sources [S1, unofficial].
*   **When to use:** Not given in the sources [S1, unofficial].
*   **When not to use:** Not given in the sources [S1, unofficial].

## Background
3DGS-DR was introduced to address the limitations of the emission-absorption model in standard 3DGS, which bakes lighting into spherical harmonics and prevents relighting [S1, unofficial]. It is part of a family of methods including GaussianShader and IRGS that incorporate classical reflection and shading concepts into the 3DGS framework [S1, unofficial]. The method builds on the 3D Gaussian Splatting foundation [Parent](1174-3d-gaussian-splatting-with-brdf-embedding.md) [S1, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
[S2] ESVR: 3D Ellipsoid-based Sparse Volume Rendering via Structure-aware Primitive Learning and Per-primitive Ray Sampling, https://arxiv.org/abs/2608.05564v1, fetched 2026-10-26 (unofficial)
