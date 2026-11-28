# Relightable Gaussian Codec Avatars

Also called: GHG, GoHA, GoHA / GHG, Relightable Gaussian head methods using FLAME rig

## Overview
Relightable Gaussian Codec Avatars (GHG) is a method for reconstructing 3D head avatars that supports dynamic relighting under novel illumination conditions [S2, unofficial]. It distinguishes itself from standard Gaussian Avatar methods by using a FLAME-rigged Gaussian representation and a relightable BRDF to enable high-fidelity synthesis under arbitrary lighting [S1, unofficial].

## How it works
**Inputs:** Video capture of a talking head; initial FLAME mesh fit via MICA or DECA [S1, unofficial].
**Outputs:** A set of 3D Gaussians with position, covariance, opacity, and material parameters for a relightable BRDF [G].

**Data Structures:**
1. **Gaussian Primitives:** Each Gaussian $i$ stores position $\mathbf{p}_i$, covariance $\Sigma_i$, opacity $\alpha_i$, and BRDF parameters (albedo, roughness) [G].
2. **FLAME Rig:** Parametric head model with pose $\theta$, expression $\beta$, and identity $\gamma$ [S1, unofficial].
3. **Deformation Network:** A lightweight MLP or explicit offset field mapping canonical positions to deformed positions conditioned on pose and expression [S1, unofficial].

**Steps:**
1. **Initialization:** Fit a FLAME mesh to the input video to obtain identity $\gamma$ [S1, unofficial]. Initialize one Gaussian per triangle of the FLAME mesh in the canonical rest pose, stored in tangent space [S1, unofficial].
2. **Deformation:** For a target pose $\theta$ and expression $\beta$, compute the rigid transformation from FLAME [S1, unofficial]. Apply pose-conditioned offsets $\Delta \mathbf{p}$ predicted by the deformation network to correct non-rigid skin deformation [S1, unofficial]. Update covariance $\Sigma_i$ using the Jacobian $J$ of the deformation map: $\Sigma'_i = J \Sigma_i J^T + \sigma^2 I$ [G].
3. **Relightable Rendering:** Project deformed Gaussians to screen space and accumulate contributions front-to-back [S1, unofficial]. Compute color using a relightable BRDF $f_r(\mathbf{v}, \mathbf{l}, \mathbf{n}, \mathbf{p})$ parameterized by view $\mathbf{v}$, light $\mathbf{l}$, normal $\mathbf{n}$, and material $\mathbf{p}$ [S1, unofficial]. This replaces static spherical harmonics to allow rendering under arbitrary light environments $\mathbf{L}$ [S2, unofficial].
4. **Optimization:** Minimize photometric $\mathcal{L}_{L1}$ and perceptual $\mathcal{L}_{MS-SSIM}$ losses between rendered and ground truth images [S1, unofficial]. Freeze geometry and train the relightable BRDF parameters in a second stage if needed [S1, unofficial].

**Parameters:** Not given in the sources.
**Edge Cases:** Not given in the sources.

## Subtopics

### [iVR-GS Relighting via Gaussian Primitives](7018-ivr-gs-relighting-via-gaussian-primitives.md)
Focuses on volume visualization scenes with composable primitives and Blinn-Phong relighting, distinct from GHG's head-specific FLAME rig and BRDF approach.

## Operating characteristics
* **Memory:** Compresses to $\le 30$ MB for a full avatar, including BRDF parameters [S1, unofficial].
* **Strengths:** Supports diverse lighting conditions and dynamic relighting [S2, unofficial].
* **Limitations:** Relighting quality depends on BRDF accuracy and FLAME fit quality [S1, unofficial].

## Background
Relightable Gaussian Codec Avatars were introduced to enable dynamic lighting changes for virtual production and telepresence [S2, unofficial]. The method builds on [Gaussian Avatar Reconstruction](1575-gaussian-avatar-reconstruction.md) principles, specifically using the FLAME parametric head model for rigging [S1, unofficial].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
