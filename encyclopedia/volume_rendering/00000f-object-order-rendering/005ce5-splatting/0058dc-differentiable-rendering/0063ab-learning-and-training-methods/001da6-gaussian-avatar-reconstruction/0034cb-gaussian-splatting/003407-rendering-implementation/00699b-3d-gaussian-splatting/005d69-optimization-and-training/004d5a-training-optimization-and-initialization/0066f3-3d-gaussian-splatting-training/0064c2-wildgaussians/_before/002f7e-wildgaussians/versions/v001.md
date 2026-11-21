# WildGaussians

Also called: 3D Reconstruction In the Wild (WildGaussians)

## Overview
WildGaussians is a method for "in-the-wild" 3D reconstruction that integrates robust DINO features and a lightweight appearance modeling module within the 3D Gaussian Splatting (3DGS) framework to handle unconstrained photo collections [S1][S2, unofficial]. It sets itself apart from standard 3DGS by explicitly addressing varying illumination, occlusions, and transient objects found in real-world data, while maintaining real-time rendering speeds [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** An unconstrained collection of images (potentially with varying lighting, occlusions, and transients) and their associated camera poses.
*   **Outputs:** A set of 3D Gaussians optimized to represent the scene, including parameters for position, covariance, opacity, spherical harmonics (color), and appearance features.

**Data Structures:**
The method extends the standard 3DGS Gaussian representation. Each Gaussian $i$ retains standard attributes: mean position $\mu_i$, covariance matrix $\Sigma_i$, opacity $\alpha_i$, and spherical harmonics coefficients $\mathbf{c}_i$. Additionally, it introduces:
1.  **DINO Features:** Robust semantic features extracted from a pre-trained DINO model, likely associated with the Gaussians to provide semantic consistency and robustness against photometric variations [S1][S2, unofficial].
2.  **Appearance Modeling Module:** A lightweight module that captures local appearance variations (lighting, shadows) that global embeddings miss. This likely involves per-Gaussian or local feature vectors that are modulated by view-dependent appearance features [S1][S2, unofficial].

**Steps:**
1.  **Initialization:** Initialize 3D Gaussians from the input images, typically using Structure-from-Motion (SfM) points or a similar initialization strategy, as per the base 3DGS framework.
2.  **Feature Extraction:** Extract DINO features from the input images. These features serve as robust semantic anchors [S1][S2, unofficial].
3.  **Appearance Feature Modeling:** Implement a lightweight appearance modeling module. This module allows each Gaussian to sample from multiple feature maps or maintain local appearance features, capturing details like highlights and shadows that global scene embeddings fail to represent [S1][S2, unofficial].
4.  **Differentiable Rendering:** Render the scene using differentiable Gaussian splatting. The rendering process incorporates the DINO features and the appearance-modulated color/opacity. The unscented transform may be used for more precise Gaussian projection onto the image plane [S1][S2, unofficial].
5.  **Loss Computation:** Compute losses based on:
    *   Photometric reconstruction loss (e.g., L1, SSIM) between rendered and input images.
    *   DINO feature consistency loss, ensuring semantic alignment between rendered features and input image features [S1][S2, unofficial].
    *   Appearance consistency losses, if applicable, to handle varying illumination [S1][S2, unofficial].
6.  **Optimization:** Optimize the Gaussian parameters (position, covariance, opacity, SH coefficients, appearance features) using stochastic gradient descent (e.g., Adam). The optimization handles unconstrained photo collections by being robust to occlusions and transients [S1][S2, unofficial].
7.  **Densification/Pruning:** Apply standard 3DGS densification and pruning strategies to maintain an efficient representation [S1][S2, unofficial].

**Formulas:**
*   **Rendering:** Standard 3DGS compositing formula, modified to include appearance features.
*   **Loss:** $\mathcal{L} = \lambda_{img}\mathcal{L}_{img} + \lambda_{DINO}\mathcal{L}_{DINO} + \lambda_{app}\mathcal{L}_{app}$, where $\mathcal{L}_{img}$ is photometric loss, $\mathcal{L}_{DINO}$ is DINO feature loss, and $\mathcal{L}_{app}$ is appearance consistency loss. Coefficients $\lambda$ are hyperparameters.

**Parameters:**
*   Learning rates for Gaussian parameters and appearance features.
*   Loss weights $\lambda_{img}, \lambda_{DINO}, \lambda_{app}$.
*   DINO feature dimensionality.
*   Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Transient Objects:** The method aims to mitigate transient effects, but severe transients may still cause artifacts if not properly handled by the appearance modeling or visibility maps (though visibility maps are more explicitly mentioned for GS-W [S1][S2, unofficial]).
*   **Varying Illumination:** The appearance modeling module is crucial; if insufficient, lighting changes may cause inconsistent reconstructions.
*   **Precision:** The use of the unscented transform for projection may add computational overhead but improves accuracy [S1][S2, unofficial].

## Subtopics

## Operating characteristics
*   **Time Complexity:** Matches real-time rendering speeds of standard 3DGS [S1][S2, unofficial]. Training time is likely higher than standard 3DGS due to the additional appearance modeling and DINO feature loss computation.
*   **Memory Complexity:** Higher than standard 3DGS due to the storage of DINO features and appearance features per Gaussian.
*   **Measured Performance:** Outperforms the original NeRF baseline on wild data and has similar speed to 3DGS on wild data [S1][S2, unofficial].
*   **Strengths:** Handles unconstrained photo collections, varying illumination, occlusions, and transient objects. Maintains real-time rendering speeds.
*   **Limitations:** Increased memory usage. Performance depends on the quality of DINO features and the effectiveness of the appearance modeling module.
*   **When to use:** When reconstructing scenes from unconstrained, real-world photo collections with varying lighting and potential transients.
*   **When not to use:** When real-time rendering speed is critical and memory is constrained, or when the scene is controlled and well-lit (standard 3DGS may suffice).

## Background
WildGaussians was introduced as part of the effort to extend 3D Gaussian Splatting to "in-the-wild" settings [S1][S2, unofficial]. It addresses the limitations of standard 3DGS, which assumes controlled environments, by integrating robust DINO features and a lightweight appearance modeling module [S1][S2, unofficial]. It is compared with other "in-the-wild" methods like Gaussian in the Wild (GS-W) and Wild-GS [S1][S2, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
