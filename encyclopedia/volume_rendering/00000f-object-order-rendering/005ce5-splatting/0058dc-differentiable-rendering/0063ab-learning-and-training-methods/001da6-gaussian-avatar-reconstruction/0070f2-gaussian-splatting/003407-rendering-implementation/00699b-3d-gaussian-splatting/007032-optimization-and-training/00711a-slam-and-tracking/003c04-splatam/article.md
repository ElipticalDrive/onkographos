# SplaTAM

Also called: SplaTAM System
## Overview
SplaTAM is a dense RGB-D SLAM system that maps the world as a collection of 3D Gaussians, using depth and photometric ICP to track and densify online [S2, S3, unofficial]. It distinguishes itself from sibling methods by using view-independent colors for Gaussians and a specific densification mask to prevent duplicate densification, achieving high fidelity on the Replica dataset [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** RGB-D image sequences [S2, unofficial].
*   **Output:** A dense 3D Gaussian map $\mathcal{G}$ and camera poses $T_{CW}$ [S2, unofficial].

**Data Structures:**
The scene is represented as a set of $N$ 3D Gaussians $\mathcal{G} = \{g_1, \dots, g_N\}$. Each Gaussian $g_i$ is defined by:
*   Mean position $\boldsymbol{\mu}_i \in \mathbb{R}^3$ [G].
*   Covariance matrix $\boldsymbol{\Sigma}_i \in \mathbb{R}^{3 \times 3}$, parameterized by rotation $R_i \in SO(3)$ and scale $S_i \in \mathbb{R}^3$ [G].
*   Opacity $\alpha_i \in [0, 1]$ [G].
*   View-independent color $c_i$ [S1]. Note: Unlike standard 3DGS, SplaTAM uses view-independent colors [S1].

**Algorithm Steps:**
1.  **Initialization:** Initialize the Gaussian map from the first RGB-D frame [G].
2.  **Tracking:** Optimize the camera pose $T_{CW}$ for the current frame by minimizing the reprojection error against the existing Gaussian map, using depth and photometric ICP [S3, unofficial].
3.  **Densification Check:** Create a densification mask for the current frame. This mask determines if a pixel needs densification by considering the current Gaussians and the captured depth of the new frame [S1]. This step avoids duplicate densification [S1].
4.  **Mapping:** If densification is required, add new Gaussians based on the mask. Optimize the Gaussian parameters (position, scale, rotation, opacity, color) and the camera pose jointly to minimize the photometric and geometric loss [S3, unofficial].
5.  **Pruning:** Remove unreliable Gaussians based on low opacity or large error [G].

**Formulas:**
The optimization minimizes a loss function $L$:
$$ L = L_{rgb} + \lambda L_{depth} + L_{reg} $$
where $L_{rgb}$ is the photometric loss, $L_{depth}$ is the depth loss, and $L_{reg}$ is regularization [S1].

**Parameters:**
*   Typical values for Replica dataset performance: PSNR 34.11, SSIM 0.97, LPIPS 0.10 [S1].

**Edge Cases:**
*   Duplicate densification is mitigated by the densification mask [S1].
*   Localization stability is improved by the joint optimization of pose and map [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Online, real-time capable on standard hardware [S3, unofficial].
*   **Memory Complexity:** Scales with the number of Gaussians $N$ [G].
*   **Performance:** On the Replica dataset, SplaTAM achieves PSNR 34.11, SSIM 0.97, and LPIPS 0.10 [S1].
*   **Strengths:** Dense RGB-D input allows for accurate depth integration; view-independent colors simplify the optimization [S1].
*   **Limitations:** Requires RGB-D input, unlike monocular methods [S3, unofficial].

## Background
SplaTAM was introduced by Nikhil Keetha, Jay Karhade, Krishna Murthy Jatavallabhula, Gengshan Yang, Sebastian Scherer, Deva Ramanan, and Jonathon Luiten at CVPR 2024 [S2, unofficial]. It is part of the broader effort to integrate 3D Gaussian Splatting into SLAM systems, providing a dense, photorealistic map that can be used for tracking and mapping simultaneously [S3, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] spla-tam/SplaTAM, https://github.com/spla-tam/SplaTAM, fetched 2026-10-27 (unofficial)
[S3] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
