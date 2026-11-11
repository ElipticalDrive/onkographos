# SplaTAM

Also called: SplaTAM: RGB-D SLAM with 3D Gaussian Splatting

## Overview
SplaTAM is an online RGB-D SLAM method that represents the scene as a collection of 3D Gaussians, enabling simultaneous localization and mapping [S2, unofficial]. It distinguishes itself from sibling methods like Gaussian-SLAM by using view-independent colors and a specific densification mask to prevent duplicate Gaussian creation, relying on depth and photometric ICP for tracking [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** RGB-D image pairs $(I_t, D_t)$ at time $t$.
*   **Outputs:** Camera pose $\mathbf{T}_t$, updated set of 3D Gaussians $\mathcal{G}$.

**Data Structures:**
*   **Gaussians $\mathcal{G}$:** Each Gaussian $i$ has mean $\mu_i$, covariance $\Sigma_i$ (parameterized by scale $\mathbf{s}_i$ and rotation $\mathbf{R}_i$), opacity $\alpha_i$, and view-independent color $c_i$ [S1].
*   **Densification Mask:** A binary mask indicating pixels in the current frame that require new Gaussians, determined by comparing current Gaussians against captured depth [S1].

**Steps:**
1.  **Initialization:** Initialize Gaussians from the first frame's depth or point cloud [G].
2.  **Tracking:** Estimate camera pose $\mathbf{T}_t$ using Iterative Closest Point (ICP) on depth data and photometric optimization [S2, unofficial].
3.  **Rendering:** Render the current view using differentiable rasterization of the current Gaussian set [G].
4.  **Densification:** Determine which pixels need densification using the densification mask, which considers the current Gaussians and the captured depth of the new frame [S1]. Add new Gaussians only in these regions to avoid duplicates [S1].
5.  **Optimization:** Update Gaussian parameters and camera poses to minimize the difference between rendered and captured data [G].

**Loss Function:**
The total loss $L$ combines photometric and geometric terms:
$$ L = L_{photo} + \lambda L_{depth} + L_{reg} $$
where $L_{photo} = || I_{rendered} - I_{captured} ||_1$ and $L_{depth} = || D_{rendered} - D_{captured} ||_1$ [G].

**Parameters:**
*   $\lambda$: Weight for depth loss. Not given in the sources.

**Edge Cases:**
*   **Duplicate Densification:** Avoided by using view-independent colors and the densification mask [S1].
*   **Map Drift:** Mitigated by the online tracking loop [G].

## Operating characteristics
*   **Time Complexity:** Online, suitable for real-time applications [S2, unofficial].
*   **Memory Complexity:** Scales with the number of Gaussians, which grows with scene size and time [G].
*   **Performance:** On the Replica dataset, SplaTAM achieves PSNR 34.11, SSIM 0.97, and LPIPS 0.10 [S1].
*   **Strengths:** Real-time performance, high-quality reconstruction, online mapping [S2, unofficial].
*   **Limitations:** Requires RGB-D input [S2, unofficial].

## Background
SplaTAM was introduced as an RGB-D SLAM method that maps the world as 3D Gaussian Splatting [S2, unofficial]. It is part of the broader family of SLAM methods using 3D Gaussians, which also includes Gaussian-SLAM and Gaussian Splatting SLAM [S2, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
