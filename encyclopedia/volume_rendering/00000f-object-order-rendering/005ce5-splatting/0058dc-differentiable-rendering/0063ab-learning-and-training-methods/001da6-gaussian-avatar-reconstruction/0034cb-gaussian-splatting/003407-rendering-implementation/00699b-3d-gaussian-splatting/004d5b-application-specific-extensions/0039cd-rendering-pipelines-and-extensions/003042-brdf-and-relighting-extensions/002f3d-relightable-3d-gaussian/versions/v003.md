# Relightable 3D Gaussian

Also called: Relightable 3D Gaussian Splatting

## Overview
Relightable 3D Gaussian is a method that extends 3D Gaussian Splatting by assigning per-Gaussian Bidirectional Reflectance Distribution Function (BRDF) parameters and precomputed visibility to enable dynamic relighting and shadow casting [S1, unofficial]. It distinguishes itself from standard 3D Gaussian Splatting by decoupling material properties from view-dependent appearance, allowing the scene to be illuminated by arbitrary light sources rather than relying solely on captured environment maps [S1, unofficial].

## How it works
**Inputs:**
*   A set of 3D Gaussians $\{G_i\}$, where each Gaussian $i$ is defined by mean $\mu_i$, covariance $\Sigma_i$, opacity $\alpha_i$, and spherical harmonic coefficients $c_{i, l, m}$ for view-dependent color [G].
*   A scene graph or light source definition (directional, point, or area lights).

**Outputs:**
*   A rendered image with physically plausible shading, including shadows and specular reflections based on the defined BRDF.

**Data Structures:**
Each Gaussian $i$ stores the following fields:
1.  **Geometry:** $\mu_i \in \mathbb{R}^3$ (position), $\Sigma_i \in \mathbb{R}^{3 \times 3}$ (covariance), $\alpha_i \in [0, 1]$ (opacity).
2.  **Material (BRDF):**
    *   Albedo $\rho_i \in \mathbb{R}^3$ (diffuse color).
    *   Roughness $r_i \in [0, 1]$ (microfacet roughness).
    *   Normal $n_i \in \mathbb{R}^3$ (surface normal, often derived from the Gaussian's orientation or learned).
3.  **Visibility:** A precomputed visibility field or shadow map contribution $v_i$, representing the Gaussian's occlusion status relative to light sources [S1, unofficial].

**Steps:**
1.  **Initialization:** Initialize Gaussians from a sparse point cloud or dense reconstruction. Assign initial BRDF parameters ($\rho_i, r_i, n_i$) and visibility $v_i$.
2.  **Optimization:** Optimize the Gaussian parameters using differentiable rendering. The loss function typically includes:
    *   Photometric loss between rendered and ground-truth images under known lighting.
    *   Regularization on BRDF parameters to ensure physical plausibility.
    *   Visibility optimization to capture shadow boundaries [S1, unofficial].
3.  **Rendering:**
    *   For each pixel, project the 3D Gaussians to screen space.
    *   Sort Gaussians by depth.
    *   For each Gaussian, compute the BRDF response based on the incoming light direction $L$, view direction $V$, and normal $n_i$.
    *   Apply the precomputed visibility $v_i$ to modulate the contribution (shadowing).
    *   Composite the colors using front-to-back alpha blending.

**Formulas:**
The color contribution $C_i$ of a Gaussian $i$ is computed as:
$$ C_i = \rho_i \cdot f_r(L, V, n_i, r_i) \cdot v_i $$
where $f_r$ is the BRDF model (e.g., Cook-Torrance) [S1, unofficial].

**Parameters:**
*   Roughness range: $[0, 1]$ [S1, unofficial].
*   Albedo range: $[0, 1]^3$ [S1, unofficial].

**Edge Cases:**
*   **Degenerate Normals:** If the normal $n_i$ is not well-defined, shading artifacts occur. [G] Ensure normals are normalized.
*   **Visibility Precision:** Precomputed visibility may suffer from aliasing at shadow boundaries. [G] Use soft shadows or higher resolution visibility maps if available.

## Subtopics

### R3DG
R3DG is a variant that combines BRDF parameters with ambient occlusion baked per Gaussian [S1, unofficial]. It simplifies the lighting model by incorporating indirect lighting effects directly into the Gaussian's material properties, reducing the need for complex global illumination calculations during rendering. Choose this when real-time performance is critical and indirect lighting approximations are sufficient.

## Operating characteristics
*   **Time Complexity:** Rendering time is similar to standard 3D Gaussian Splatting, dominated by rasterization and BRDF evaluation per Gaussian. Optimization time is higher due to the additional parameters (BRDF, visibility).
*   **Memory Complexity:** Increased memory usage per Gaussian to store albedo, roughness, normal, and visibility data.
*   **Strengths:** Enables dynamic relighting, physically based materials, and shadow casting.
*   **Limitations:** Requires careful optimization of BRDF parameters; precomputed visibility may not capture complex inter-reflections.
*   **When to use:** Scenes requiring dynamic lighting changes or physically based rendering.
*   **When not to use:** Static scenes with simple lighting where standard 3D Gaussian Splatting suffices.

## Background
Relightable 3D Gaussian was introduced by Gao et al. in ECCV'24 [S1, unofficial]. It builds on the success of 3D Gaussian Splatting by addressing its limitation in handling dynamic lighting and physically based materials. The method has been used in applications requiring interactive relighting and realistic material representation.

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
