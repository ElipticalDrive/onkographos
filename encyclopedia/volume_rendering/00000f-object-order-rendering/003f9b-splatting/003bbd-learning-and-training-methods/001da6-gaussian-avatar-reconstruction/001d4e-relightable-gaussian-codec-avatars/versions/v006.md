# Relightable Gaussian Codec Avatars

Also called: GHG, GoHA, GoHA / GHG, Relightable Gaussian head methods using FLAME rig

## Overview
Relightable Gaussian Codec Avatars is a method for creating photorealistic, animatable 3D head avatars that support dynamic relighting under diverse lighting conditions [S2, unofficial]. It builds on [Gaussian Avatar Reconstruction](1575-gaussian-avatar-reconstruction.md) by integrating a relightable appearance model with a FLAME-based parametric rig [S1, unofficial]. Unlike standard Gaussian Avatars which assume fixed lighting, this method explicitly models the interaction between light and surface geometry, allowing for high-fidelity synthesis under novel illumination [S2, unofficial].

## How it works
**Inputs:**
*   Multi-view video of a subject (typically ~30 seconds) [S1, unofficial].
*   Camera poses and intrinsics.
*   FLAME parametric head model parameters (shape $\beta$, expression $\alpha$, pose $\theta$) estimated via fitting tools like MICA or DECA [S1, unofficial].

**Outputs:**
*   A set of 3D Gaussians $\{g_i\}$ representing the head volume.
*   A relightable BRDF or appearance network parameters.
*   Rendered images under arbitrary lighting environments.

**Data Structures:**
1.  **Gaussian Primitives:** Each Gaussian $i$ is defined by:
    *   Position $\mathbf{p}_i \in \mathbb{R}^3$: Constrained to the FLAME mesh surface.
    *   Covariance $\Sigma_i \in \mathbb{R}^{3 \times 3}$: Determined by orientation and scale.
    *   Opacity $\alpha_i \in [0, 1]$.
    *   Appearance: Stored as coefficients for a relightable model (e.g., spherical harmonics for diffuse/specular components or a small MLP) [S1, unofficial].
2.  **FLAME Mesh:** A template mesh with $V$ vertices and $F$ triangles.
3.  **Deformation MLP:** A small network mapping pose/expression parameters to per-Gaussian offsets [S1, unofficial].

**Steps:**
1.  **Initialization:**
    *   Fit the FLAME model to the input video to obtain initial shape, expression, and pose parameters [S1, unofficial].
    *   Initialize one Gaussian per triangle of the FLAME mesh.
    *   Set Gaussian positions to the triangle centroids or vertices.
    *   Orient Gaussians along the mesh surface normals.
    *   Initialize appearance coefficients to match the input lighting [S1, unofficial].
2.  **Deformation:**
    *   For a given frame with pose $\theta$ and expression $\alpha$, compute the deformed FLAME mesh vertices $\mathbf{V}'(\theta, \alpha)$ [S1, unofficial].
    *   Update Gaussian positions $\mathbf{p}_i$ by mapping them to the deformed mesh surface. This can be done via barycentric interpolation or a pose-conditioned offset MLP [S1, unofficial].
3.  **Relightable Appearance Modeling:**
    *   Instead of static spherical harmonics, decompose appearance into view-independent diffuse/albedo components and view-dependent specular components [S2, unofficial].
    *   The diffuse component is often modeled as a constant color or low-order SH, while the specular component uses higher-order SH or a neural radiance field (NeRF) style MLP conditioned on view and light direction [S1, unofficial].
4.  **Rendering:**
    *   Project Gaussians to 2D using the camera matrix.
    *   Sort Gaussians by depth.
    *   Composite colors using alpha blending:
        $$ C = \sum_{i \in N} c_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
        where $c_i$ is the color of Gaussian $i$ under the current lighting environment [S1, unofficial].
5.  **Training:**
    *   Optimize Gaussian parameters (positions, covariances, opacities, appearance coefficients) and the deformation network weights.
    *   Loss function $\mathcal{L}$ combines:
        *   Photometric loss: $\mathcal{L}_{L1}$ or $\mathcal{L}_{MSE}$ between rendered and ground truth images [S1, unofficial].
        *   Perceptual loss: $\mathcal{L}_{MS-SSIM}$ or $\mathcal{L}_{LPIPS}$ [S1, unofficial].
        *   Regularization: Smoothness on Gaussian parameters [S1, unofficial].
    *   Freeze geometry after initial convergence and train the relightable BRDF parameters if needed [S1, unofficial].

**Parameters:**
*   Number of Gaussians: Typically one per triangle or vertex of the FLAME mesh (e.g., ~50k-100k) [S1, unofficial].
*   Compression: Target size $\le 30$ MB [S1, unofficial].

**Edge Cases:**
*   **Occlusion:** Self-occlusion is handled by the depth sorting in splatting.
*   **Lighting Extremes:** Extreme lighting may require careful normalization of BRDF inputs to avoid saturation [S2, unofficial].
*   **Precision:** Floating-point precision in position updates during deformation can cause jitter; use stable interpolation methods [G].

## Subtopics

### [iVR-GS Relighting via Gaussian Primitives](7018-ivr-gs-relighting-via-gaussian-primitives.md)
iVR-GS is a novel view synthesis (NVS) framework for volume visualization (VolVis) scenes that leverages composable and editable 3D Gaussian primitives to enable flexible exploration without direct access to volume data. It distinguishes itself from baseline 3D Gaussian Splatting (3DGS) and NeRF-based methods (Plenoxels, CCNeRF) by explicitly modeling lighting effects via the Blinn-Phong model, allowing for high-quality relighting and scene editing. Unlike simple image-space interpolation, iVR-GS reconstructs underlying geometry to ensure physically plausible relighting results.

## Operating characteristics
*   **Time Complexity:** Training time is comparable to [Gaussian Avatars](1477-gaussian-avatars.md) but may be longer due to the additional complexity of the relightable appearance model [S2, unofficial]. Rendering is real-time on modern GPUs, similar to standard 3DGS [S1, unofficial].
*   **Memory Complexity:** Storage is compact, typically $\le 30$ MB for the avatar model [S1, unofficial].
*   **Strengths:** High-fidelity relighting, real-time rendering, compact representation, compatibility with standard avatar rigs (FLAME) [S1, unofficial][S2, unofficial].
*   **Limitations:** Requires multi-view input for initial fitting; performance depends on the quality of the FLAME fit; may struggle with extreme non-Lambertian materials if not explicitly modeled [S1, unofficial][S2, unofficial].
*   **When to Use:** Applications requiring dynamic lighting changes, such as virtual try-on, telepresence, or interactive avatars [S2, unofficial].
*   **When Not to Use:** Single-view reconstruction without strong priors; scenes with complex, non-head geometry [S1, unofficial].

## Background
Relightable Gaussian Codec Avatars was introduced to address the limitations of previous Gaussian avatar methods that assumed fixed lighting [S2, unofficial]. It builds on the foundation of [Gaussian Avatar Reconstruction](1575-gaussian-avatar-reconstruction.md) and integrates concepts from relightable rendering and parametric head models (FLAME) [S1, unofficial]. The method represents a step towards more versatile and interactive 3D human avatars [S2, unofficial].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
