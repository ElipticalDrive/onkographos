# Relightable Gaussian Codec Avatars

Also called: GHG, GoHA, GoHA / GHG, Relightable Gaussian head methods using FLAME rig

## Overview
Relightable Gaussian Codec Avatars (GHG) is a method for reconstructing photorealistic, animatable 3D head avatars from monocular video that supports dynamic relighting under novel illumination conditions [S2, unofficial]. It distinguishes itself from standard Gaussian Avatar methods by decoupling geometry and appearance into a FLAME-rigged Gaussian representation and a relightable BRDF, enabling high-fidelity synthesis under arbitrary lighting [S1, unofficial]. Unlike [GaussianShader](1470-gaussianshader.md) which often uses learned neural radiance fields or simpler shading, GHG explicitly models surface reflectance properties to allow interactive relighting [S1, unofficial].

## How it works
**Inputs:** Monocular video of a talking head; initial FLAME mesh fit (via MICA or DECA).
**Outputs:** A set of 3D Gaussians with position, covariance, opacity, and spherical harmonic coefficients for appearance, plus a relightable BRDF model.

**Data Structures:**
1.  **Gaussian Primitives:** Each Gaussian $i$ stores:
    *   Position $\mathbf{p}_i \in \mathbb{R}^3$ (anchored to FLAME mesh surface).
    *   Covariance $\Sigma_i \in \mathbb{R}^{3 \times 3}$ (parameterized by rotation $\mathbf{R}_i$ and scale $\mathbf{s}_i$).
    *   Opacity $\alpha_i \in [0, 1]$.
    *   Appearance: Spherical Harmonics (SH) coefficients $\mathbf{c}_i$ for base color, and parameters for the BRDF (e.g., albedo, roughness, normal).
2.  **FLAME Rig:** Parametric head model with pose $\theta$, expression $\beta$, and identity $\gamma$.
3.  **Deformation Network:** A small MLP or explicit offset field that maps canonical Gaussian positions to deformed positions based on pose and expression.

**Steps:**
1.  **Initialization:** Fit a FLAME mesh to the input video frames to obtain identity $\gamma$. Initialize one Gaussian per triangle (or vertex) of the FLAME mesh in the canonical rest pose. Assign initial SH coefficients from the input image colors [S1, unofficial].
2.  **Deformation:** For a target pose $\theta$ and expression $\beta$:
    *   Compute the rigid transformation $T(\theta, \beta)$ from FLAME.
    *   Apply pose-conditioned offsets $\Delta \mathbf{p}$ predicted by a lightweight network or stored as explicit offsets to correct for non-rigid skin deformation [S1, unofficial].
    *   Deform covariance $\Sigma$ using the Jacobian of the deformation field to maintain volume preservation [S1, unofficial].
3.  **Relightable Rendering:**
    *   Project deformed Gaussians to screen space.
    *   For each pixel, accumulate contributions from front-to-back sorted Gaussians.
    *   The color $C$ of a Gaussian is computed using a relightable BRDF model $f_r(\mathbf{v}, \mathbf{l}, \mathbf{n}, \mathbf{p})$ where $\mathbf{v}$ is view direction, $\mathbf{l}$ is light direction, $\mathbf{n}$ is normal, and $\mathbf{p}$ represents material parameters (albedo, roughness) [S1, unofficial].
    *   Unlike static SH, the BRDF allows the same geometry to be rendered under arbitrary light environments $\mathbf{L}$ [S2, unofficial].
4.  **Optimization:**
    *   Minimize photometric loss $\mathcal{L}_{L1}$ and perceptual loss $\mathcal{L}_{MS-SSIM}$ between rendered and ground truth images [S1, unofficial].
    *   Add regularization on Gaussian scale and opacity to prevent overfitting [S1, unofficial].
    *   If relighting is enabled, optimize BRDF parameters jointly or in a second stage after geometry convergence [S1, unofficial].

**Parameters:**
*   Learning rate: $2 \times 10^{-4}$ for Gaussians, $1 \times 10^{-3}$ for BRDF [S1, unofficial].
*   Number of Gaussians: $\sim 10^5$ to $10^6$ for high fidelity [S1, unofficial].

**Edge Cases:**
*   **Occlusions:** Monocular input lacks depth ambiguity; rely on FLAME prior to prevent floating artifacts [S1, unofficial].
*   **Lighting Mismatch:** If training lighting is unknown, assume neutral lighting or estimate it as a parameter [S1, unofficial].

## Subtopics

### [iVR-GS Relighting via Gaussian Primitives](7018-ivr-gs-relighting-via-gaussian-primitives.md)
Focuses on volume visualization scenes with composable primitives and Blinn-Phong relighting, distinct from GHG's head-specific FLAME rig and BRDF approach.

## Operating characteristics
*   **Time Complexity:** Training is comparable to [Gaussian Avatars](1477-gaussian-avatars.md) but requires additional optimization for BRDF parameters, increasing convergence time [S1, unofficial]. Rendering is real-time on modern GPUs due to splatting efficiency [S1, unofficial].
*   **Memory:** $\sim 30$ MB compressed for a full avatar, including BRDF parameters [S1, unofficial].
*   **Strengths:** High-fidelity relighting; animatable via FLAME rig; efficient rendering [S2, unofficial].
*   **Limitations:** Requires monocular video with sufficient texture; FLAME fit quality impacts geometry; relighting quality depends on BRDF accuracy [S1, unofficial].

## Background
Relightable Gaussian Codec Avatars were introduced to address the limitations of static appearance models in Gaussian Avatar methods, enabling dynamic lighting changes for virtual production and telepresence [S2, unofficial]. The method builds on [Gaussian Avatar Reconstruction](1575-gaussian-avatar-reconstruction.md) principles, specifically using the FLAME parametric head model for rigging [S1, unofficial].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
