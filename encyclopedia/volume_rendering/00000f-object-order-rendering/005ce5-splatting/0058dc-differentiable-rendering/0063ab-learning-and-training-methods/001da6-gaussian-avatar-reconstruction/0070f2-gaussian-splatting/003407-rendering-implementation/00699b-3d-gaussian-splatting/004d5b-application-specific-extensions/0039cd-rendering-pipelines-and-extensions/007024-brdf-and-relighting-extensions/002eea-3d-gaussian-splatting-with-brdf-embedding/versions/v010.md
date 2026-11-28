# 3D Gaussian Splatting with BRDF Embedding

Also called: 3D Gaussian Splatting (3DGS), Gao et al. BRDF Embedding, Gao et al. BRDF Method

## Overview
This method extends 3D Gaussian Splatting by embedding physically based rendering parameters into each Gaussian primitive, enabling per-point BRDF evaluation and dynamic relighting [S1][S2, unofficial]. It addresses the limitation of standard 3DGS, which uses an emission-absorption model that bakes lighting conditions into spherical harmonics coefficients, thereby limiting performance on highly reflective surfaces and preventing relighting [S1]. It is distinguished from sibling methods like [GaussianShader](8013-gaussianshader-material-model.md) by its specific embedding of incident lighting alongside material properties [S1][S2, unofficial].

## How it works
**Inputs:** A set of 3D Gaussians optimized from multi-view images; an environment map for direct lighting [S1][S2, unofficial].
**Outputs:** A rendered image with physically based shading [G].

**Data Structures:**
Each Gaussian $i$ stores the following fields in addition to standard 3DGS attributes (position $\mathbf{p}$, covariance $\mathbf{\Sigma}$, opacity $\alpha$, SH coefficients for indirect lighting) [G]:
1.  **Albedo** ($\mathbf{c}_d$): A 3-component vector representing diffuse color [G].
2.  **Roughness** ($\alpha_r$): A scalar value controlling specular width [G].
3.  **Surface Normal** ($\mathbf{n}$): A 3-component unit vector [G].
4.  **Incident Lighting** ($\mathbf{L}_{inc}$): Parameters or coefficients representing the incoming light field, often derived from or combined with environment maps [S1][S2, unofficial].

**Steps:**
1.  **Parameter Storage:** Embed albedo, roughness, surface normals, and incident lighting parameters into each Gaussian primitive [S1][S2, unofficial].
2.  **Direct Lighting Reconstruction:** Reconstruct direct lighting contributions using the stored environment map and the embedded BRDF parameters [S1][S2, unofficial].
3.  **Indirect Lighting Modeling:** Model indirect lighting contributions using spherical harmonics coefficients stored per Gaussian [S1][S2, unofficial].
4.  **Compositing:** Alpha-composite the shaded colors from front to back using standard 3DGS blending rules [G].

**Formulas:**
The final color $C$ for a pixel is computed as:
$$C = \sum_{i \in \text{sorted}} c_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j)$$
where $c_i = f_{\text{BRDF}}(\mathbf{c}_{d,i}, \alpha_{r,i}, \mathbf{n}_i, \mathbf{L}_{\text{direct},i}) + L_{\text{indirect},i}$ combines the embedded albedo $\mathbf{c}_{d,i}$, roughness $\alpha_{r,i}$, and normal $\mathbf{n}_i$ with direct lighting $\mathbf{L}_{\text{direct},i}$ from the environment map, while $L_{\text{indirect},i}$ represents the indirect lighting from spherical harmonics [S1][S2, unofficial][G].

**Parameters:**
*   **Environment Map Resolution:** Not given in the sources [S1][S2, unofficial].
*   **SH Degree for Indirect Lighting:** Not given in the sources [S1][S2, unofficial].

**Edge Cases:**
*   **Precision:** Standard floating-point precision is assumed for BRDF calculations [G].
*   **Boundaries:** Gaussians at object boundaries may have inaccurate normals if not properly constrained during optimization [G].

## Subtopics

### [3DGS-DR](1471-3dgs-dr.md)
Incorporates classical reflection and shading concepts into 3DGS, addressing the emission-absorption limitation of standard 3DGS. Choose when classical shading models are preferred over full PBR embedding [G].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1][S2, unofficial].
*   **Memory Complexity:** Increased storage per Gaussian for albedo, roughness, normal, and incident lighting parameters [S1][S2, unofficial].
*   **Strengths:** Enables dynamic relighting and accurate rendering of reflective surfaces [S1][S2, unofficial].
*   **Limitations:** Not given in the sources [S1][S2, unofficial].
*   **When to use:** Not given in the sources [S1][S2, unofficial].
*   **When not to use:** Not given in the sources [S1][S2, unofficial].

## Background
Originated from work by Gao et al., this method embeds BRDF parameters into Gaussians to enable per-point physically based rendering [S1][S2, unofficial]. It builds on the 3DGS framework by integrating classical reflection and shading concepts [S1][S2, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
