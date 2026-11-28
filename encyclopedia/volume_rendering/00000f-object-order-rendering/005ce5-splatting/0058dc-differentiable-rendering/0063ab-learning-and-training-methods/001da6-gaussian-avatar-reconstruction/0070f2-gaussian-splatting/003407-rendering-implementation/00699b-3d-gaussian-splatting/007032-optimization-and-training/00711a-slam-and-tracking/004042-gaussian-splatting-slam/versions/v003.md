# Gaussian Splatting SLAM

Also called: Gaussian Splatting SLAM Tracking and Mapping Optimization, GaussianSplattingSLAM, Monocular Gaussian SLAM, RGB-D SLAM with Gaussian Splatting

## Overview
Gaussian Splatting SLAM is a simultaneous localization and mapping (SLAM) method that reconstructs a scene as a collection of 3D Gaussians while estimating camera poses, building on [3D Gaussian Splatting Optimization](15143-3d-gaussian-splatting-optimization.md). It differs from siblings like [SplaTAM](9193-splatam.md) and [Gaussian-SLAM](9195-gaussian-slam.md) by explicitly minimizing a combined photometric and geometric residual for both tracking and mapping, and by applying isotropic scale regularization to stabilize localization [S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Current camera pose $T_{CW}$, current Gaussian set $\mathcal{G}$, incoming RGB-D frame (color image $I$, depth map $D$).
- **Outputs:** Updated camera pose $T_{CW}$, updated Gaussian parameters (positions, scales, colors, opacities).

**Data Structures:**
- **Gaussians ($\mathcal{G}$):** Each Gaussian $g$ has position $\boldsymbol{\mu}$, scale $\mathbf{s}$, rotation $\mathbf{q}$, color $\mathbf{c}$, opacity $\alpha$.
- **Keyframes ($\mathcal{W}$):** A set of selected keyframes used for mapping.

**Steps:**

1. **Tracking:** Estimate the current camera pose $T_{CW}$ by minimizing the loss function below.
   - Perform up to 100 iterations [S2, unofficial].
   - Terminate early if the magnitude of the pose update is less than $10^{-4}$ [S2, unofficial].
   - For monocular settings, increase the learning rate of Gaussian positions $\boldsymbol{\mu}_W$ by a factor of 10 [S2, unofficial]. For RGB-D, use standard rates (rotation: 0.003, translation: 0.001) [S2, unofficial].

2. **Mapping:** Update the Gaussian parameters and keyframe poses.
   - Perform 150 iterations for single-process implementation [S2, unofficial].
   - Minimize the mapping loss over all Gaussians $\mathcal{G}$ and keyframes $k \in \mathcal{W}$.

3. **Keyframing:** Select new keyframes based on Gaussian covisibility.
   - Calculate Intersection of Union (IOU) and Overlap Coefficient (OC) between current keyframe $i$ and candidate $j$:
     $$ IOU_{cov}(i, j) = \frac{|G_i^v \cap G_j^v|}{|G_i^v \cup G_j^v|} $$
     $$ OC_{cov}(i, j) = \frac{|\mathcal{G}_i \cap \mathcal{G}_j|}{\min(|\mathcal{G}_i^v|, |\mathcal{G}_j^v|)} $$
     where $G_i^v$ is the set of Gaussians visible in keyframe $i$ [S2, unofficial].

**Formulas:**

- **Tracking Loss:**
  $$ \min_{T \subset W \in SE(3)} \lambda_{pho} E_{pho} + (1 - \lambda_{pho}) E_{geo} $$
  where $E_{pho}$ is the photometric residual and $E_{geo}$ is the geometric residual [S2, unofficial].

- **Mapping Loss:**
  $$ \min_{\forall k \in \mathcal{W}} \sum_{\mathcal{G}, \forall k \in \mathcal{W}} (\lambda_{pho} E_{pho}^k + (1 - \lambda_{pho}) E_{geo}^k) + \lambda_{iso} E_{iso} $$
  where $E_{iso}$ is the isotropic regularization term [S2, unofficial].

**Parameters:**
- $\lambda_{pho} = 0.9$ for RGB-D experiments [S2, unofficial].
- $\lambda_{iso} = 10$ for both monocular and RGB-D experiments [S2, unofficial].

**Edge Cases and Pitfalls:**
- Monocular tracking requires higher learning rates for positions to stabilize [S2, unofficial].
- Isotropic regularization is critical for stabilizing localization and mapping [S1].

## Operating characteristics
- **Performance:** On the Replica dataset, GaussianSplattingSLAM achieves PSNR 38.90, SSIM 0.99, and LPIPS 0.07 [S1].
- **Complexity:** Not given in the sources.
- **Strengths:** High reconstruction quality; stable localization via isotropic regularization [S1].
- **Limitations:** Requires careful tuning of learning rates for monocular settings [S2, unofficial].

## Background
Gaussian Splatting SLAM was introduced in the paper "Gaussian Splatting SLAM" (arXiv:2312.06741) [S2, unofficial]. It extends 3D Gaussian Splatting to SLAM by integrating tracking and mapping modules that optimize both camera poses and Gaussian parameters simultaneously.

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Splatting SLAM, https://arxiv.org/abs/2312.06741, fetched 2026-10-27 (unofficial)
