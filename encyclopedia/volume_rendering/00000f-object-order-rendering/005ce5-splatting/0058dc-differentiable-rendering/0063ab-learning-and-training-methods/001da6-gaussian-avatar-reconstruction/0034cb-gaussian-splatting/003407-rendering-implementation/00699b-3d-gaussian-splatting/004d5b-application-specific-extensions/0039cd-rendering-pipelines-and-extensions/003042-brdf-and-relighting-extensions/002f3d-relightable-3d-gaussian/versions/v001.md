# Relightable 3D Gaussian

Also called: Relightable 3D Gaussian Splatting

## Overview
Relightable 3D Gaussian is a method that extends 3D Gaussian Splatting to support physically-based rendering by assigning per-Gaussian BRDF parameters and precomputed visibility for shadows [S1, unofficial]. It distinguishes itself from standard 3D Gaussian Splatting by enabling dynamic relighting and material variation, and from [3D Gaussian Splatting with BRDF Embedding](1174-3d-gaussian-splatting-with-brdf-embedding.md) by explicitly baking ambient occlusion and visibility data per Gaussian to handle self-shadowing [S1, unofficial].

## How it works
**Inputs:** A set of 3D Gaussians (position, covariance, opacity, view-dependent color) and a set of training images with known lighting conditions.
**Outputs:** A scene representation where each Gaussian stores BRDF parameters and visibility data.

**Data Structures:**
Each Gaussian $i$ is extended with the following fields:
1.  **Albedo** ($\rho_i$): Diffuse reflectance color.
2.  **Roughness** ($\alpha_i$): Microfacet roughness parameter.
3.  **Normal** ($\mathbf{n}_i$): Surface normal vector.
4.  **Visibility** ($V_i$): Precomputed visibility factor for shadowing.

**Steps:**
1.  **Initialization:** Initialize BRDF parameters ($\rho_i, \alpha_i, \mathbf{n}_i$) and visibility ($V_i$) for each Gaussian. [G] Typical initialization sets albedo to the mean color, roughness to 0.5, and normal to the Gaussian's principal axis.
2.  **Differentiable Rendering:** Render the scene using a differentiable rasterizer. The radiance contribution of each Gaussian is computed using a BRDF model (e.g., Cook-Torrance) combined with the precomputed visibility $V_i$ to approximate self-shadowing.
3.  **Optimization:** Minimize the difference between rendered images and ground truth images using gradient descent. The loss function typically includes photometric loss and potentially regularization terms for BRDF parameters.
4.  **Visibility Update:** Update the visibility parameter $V_i$ during optimization to account for occlusions from other Gaussians. [G] This is often done by tracking the fraction of rays blocked by other geometry during training.

**Formulas:**
The rendered color $C$ for a pixel is the sum of contributions from visible Gaussians:
$$ C = \sum_{i \in \text{sorted}} c_i \cdot \alpha_i \cdot \prod_{j < i} (1 - \alpha_j) $$
where $c_i$ is the BRDF-based color of Gaussian $i$, modulated by its visibility $V_i$. [S1, unofficial]

**Parameters:**
*   **Learning Rate:** Standard values for 3DGS optimization.
*   **BRDF Model:** Cook-Torrance or similar. [S1, unofficial]

**Edge Cases:**
*   **Degenerate Gaussians:** Ensure normals are well-defined for flat or thin Gaussians. [G]
*   **Precision:** Use float32 for BRDF parameters to avoid quantization artifacts. [G]

## Subtopics

### R3DG
R3DG is a variant that combines BRDF parameters with ambient occlusion baked per Gaussian [S1, unofficial]. It sets itself apart by explicitly baking ambient occlusion into the Gaussian representation, allowing for more accurate indirect lighting effects without complex ray tracing. Choose this variant when ambient occlusion is critical for visual fidelity. Link: [R3DG](link_to_R3DG_if_exists).

## Operating characteristics
*   **Time Complexity:** Rendering time is similar to standard 3D Gaussian Splatting, but optimization is slower due to the additional BRDF parameters and visibility updates.
*   **Memory Complexity:** Higher than standard 3DGS due to the storage of BRDF parameters and visibility data per Gaussian.
*   **Strengths:** Enables dynamic relighting, physically-based materials, and self-shadowing.
*   **Limitations:** Increased memory usage, slower optimization, potential artifacts in visibility estimation.
*   **When to Use:** Scenes requiring dynamic lighting or physically-based materials.
*   **When Not to Use:** Real-time applications with strict memory constraints or static lighting scenarios.

## Background
Relightable 3D Gaussian was introduced by Gao et al. in ECCV'24 [S1, unofficial]. It builds on the success of 3D Gaussian Splatting by adding material and lighting capabilities, addressing the limitation of standard 3DGS which only supports view-dependent color. It is used in applications requiring physically-based rendering and dynamic relighting.

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
