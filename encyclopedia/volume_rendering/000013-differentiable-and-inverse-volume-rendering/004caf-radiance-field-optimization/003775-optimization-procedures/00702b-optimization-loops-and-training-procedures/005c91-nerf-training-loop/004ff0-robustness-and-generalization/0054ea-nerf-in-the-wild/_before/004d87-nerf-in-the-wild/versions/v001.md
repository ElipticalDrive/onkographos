# NeRF in the Wild

Also called: Martin-Brualla et al, NeRF-W

## Overview
NeRF in the Wild (NeRF-W) extends standard neural radiance fields to handle unconstrained photo collections by modeling per-image appearance variations and transient objects [S1, S2, unofficial]. It addresses the limitations of original NeRF, which assumes static geometry and consistent lighting, by introducing latent embeddings for each input image [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of uncalibrated or loosely calibrated images $\{I_i\}$ with associated camera poses (extrinsics) and intrinsics.
*   **Outputs:** A radiance field that can render novel views of the static background while suppressing transient objects and adapting to lighting changes.

**Data Structures:**
1.  **Radiance Field MLP:** Takes 3D position $\mathbf{x}$, viewing direction $\mathbf{v}$, and an appearance embedding $\mathbf{c}_i$ as input. Outputs color $\mathbf{c}$ and volume density $\sigma$.
2.  **Appearance Embedding:** A learnable vector $\mathbf{c}_i \in \mathbb{R}^k$ for each image $i$. This allows the network to modulate color based on lighting conditions [S2, unofficial].
3.  **Transient Embedding:** A learnable vector $\mathbf{t}_i \in \mathbb{R}^m$ for each image $i$. This models transient objects (e.g., people, cars) that appear in some images but not others [S2, unofficial].

**Method Steps:**
1.  **Initialization:** Initialize the radiance field MLP and the appearance/transient embeddings for all input images.
2.  **Ray Sampling:** For each training image $i$, sample rays through each pixel. Sample points $\mathbf{x}_j$ along the ray using stratified sampling or importance sampling.
3.  **Feature Query:**
    *   Query the radiance field MLP with $(\mathbf{x}_j, \mathbf{v}, \mathbf{c}_i)$ to get base color $\mathbf{c}_{base}$ and density $\sigma$.
    *   The appearance embedding $\mathbf{c}_i$ is concatenated or used to modulate the MLP input/features to account for lighting variations [S2, unofficial].
4.  **Transient Modeling:**
    *   The transient embedding $\mathbf{t}_i$ is used to predict a transient opacity or color component.
    *   The final color $\mathbf{C}_i$ for a pixel is computed by volume rendering the base radiance field plus the transient component.
    *   Specifically, the transient component allows the model to "hide" pixels that contain moving objects by assigning them high transient opacity or distinct color, effectively removing them from the static background reconstruction [S2, unofficial].
5.  **Rendering:** Compute the rendered image $\hat{I}_i$ using standard volume rendering integration:
    $$ \hat{C}_i = \sum_{j=1}^N T_j \alpha_j \mathbf{c}_j $$
    where $T_j = \exp(-\sum_{k=1}^{j-1} \sigma_k \delta_k)$, $\alpha_j = 1 - \exp(-\sigma_j \delta_j)$, and $\mathbf{c}_j$ is the color at point $j$ modulated by $\mathbf{c}_i$ and $\mathbf{t}_i$.
6.  **Loss Computation:** Minimize the photometric loss between $\hat{I}_i$ and $I_i$. A distortion regularizer is often added to discourage floaters [S1, unofficial].
7.  **Optimization:** Update the MLP weights and all embeddings ($\mathbf{c}_i, \mathbf{t}_i$) using gradient descent.

**Parameters:**
*   Embedding dimension $k, m$: Not given in the sources.
*   Number of samples per ray: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Large Appearance Changes:** NeRF-W cannot handle scenes with major appearance changes (e.g., seasonal changes like snow vs. no snow) [S2, unofficial].
*   **Pose Accuracy:** Requires reasonably accurate camera poses; significant pose errors can degrade performance [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Training time is longer than standard NeRF due to the additional embeddings and the need to optimize per-image latent codes. Specific training times are not given in the sources, but the survey table lists "NeRF (original)" as taking days, implying NeRF-W is comparable or slower [S1, unofficial].
*   **Memory Complexity:** Increases by $O(N \cdot (k+m))$ where $N$ is the number of images, due to storing the appearance and transient embeddings.
*   **Strengths:** Handles unconstrained internet photos with varying lighting and transient objects [S1, S2, unofficial].
*   **Limitations:** Fails on scenes with significant appearance changes (e.g., seasonal) [S2, unofficial]. Requires camera poses [S2, unofficial].

## Background
*   **Origin:** Proposed by Martin-Brualla et al. in "NeRF in the Wild: Neural Radiance Fields for Unconstrained Photo Collections" (CVPR 2021) [S2, unofficial].
*   **History:** It was one of the first methods to address the robustness of NeRF to real-world photo collections, introducing the concept of per-image latent codes for appearance and transient modeling [S1, S2, unofficial].
*   **Usage:** Used for synthesizing novel views from internet photo collections of landmarks or scenes with dynamic elements [S1, S2, unofficial].

## Sources
[S1] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
