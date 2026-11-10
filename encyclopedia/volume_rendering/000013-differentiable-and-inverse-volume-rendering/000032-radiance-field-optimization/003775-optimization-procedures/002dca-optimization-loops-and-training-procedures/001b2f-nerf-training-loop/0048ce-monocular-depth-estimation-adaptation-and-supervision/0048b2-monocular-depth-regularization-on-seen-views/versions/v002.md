# Monocular Depth Regularization on Seen Views

Also called: Patch-wise depth alignment, Seen-view depth distillation

## Overview
This method applies a consistency loss between NeRF-rendered depth and pseudo-depth from a pre-trained monocular depth estimator (MDE) specifically for training views (seen views) [S1, unofficial]. It differs from global scale/shift fitting by computing alignment parameters per image patch, which prevents dominant objects from skewing the depth alignment and improves local geometric accuracy [S1, unofficial]. It builds on the shared principles of depth rendering and MDE extraction [Parent].

## How it works
**Inputs:**
- Set of seen view images $S = \{I_i\}$.
- Pre-trained MDE network $G_\phi$ with parameters $\phi$ [S1, unofficial].
- NeRF model capable of rendering depth $\bar{D}_i$ for camera pose corresponding to $I_i$ [S1, unofficial].
- Patch definition: A set of pixels $P$ within a local patch. The specific patch size or tiling strategy is not given in the sources [G].

**Outputs:**
- Loss value $L_{seen}$ used to update NeRF parameters.

**Data Structures:**
- $D^*_i$: Pseudo-depth map from MDE, same resolution as $I_i$.
- $\bar{D}_i$: Rendered depth map from NeRF, same resolution as $I_i$.
- $w_i, q_i$: Scalar scale and shift parameters for a specific patch.

**Steps:**
1. For each seen view image $I_i \in S$:
    a. Render the NeRF depth map $\bar{D}_i$ for the camera pose of $I_i$ [S1, unofficial].
    b. Compute the pseudo-depth map $D^*_i = G_\phi(I_i)$ using the pre-trained MDE network [S1, unofficial].
    c. Divide the image into patches (or iterate over a set of patches $P$). For each patch:
        i. Identify the set of pixels $P$ in the current patch.
        ii. Compute the scale $w_i$ and shift $q_i$ via least squares fitting between the pseudo-depth and rendered depth values within the patch. The sources cite least squares [39] but do not provide the explicit closed-form solution or iterative solver details [S1, unofficial].
        iii. Compute the per-patch loss contribution: $\sum_{p \in P} \| (w_i \cdot \text{sg}(D^*_i(p)) + q_i) - \bar{D}_i(p) \|$.
    d. Sum the losses over all patches and all seen views to get $L_{seen}$.

**Formula:**
The total loss is defined as [S1, unofficial]:

$$
L_{seen} = \sum_{I_i \in S} \sum_{p \in P} \| (w_i \cdot \text{sg}(D^*_i(p)) + q_i) - \bar{D}_i(p) \|
$$

Where:
- $S$ is the set of seen view images.
- $P$ is the set of pixels within a patch.
- $D^*_i(p)$ is the pseudo-depth value at pixel $p$ for image $I_i$.
- $\bar{D}_i(p)$ is the NeRF-rendered depth value at pixel $p$.
- $w_i$ and $q_i$ are the scale and shift parameters for the patch, computed via least squares [S1, unofficial].
- $\text{sg}(\cdot)$ is the stop-gradient operator, preventing gradients from flowing into the MDE network $G_\phi$ [S1, unofficial].
- $\| \cdot \|$ denotes the norm (typically $L_1$ or $L_2$, not explicitly specified in the source formula notation, though standard in such contexts) [S1, unofficial].

**Parameters:**
- $\text{sg}(\cdot)$: Stop-gradient operation [S1, unofficial].
- $w_i, q_i$: Scale and shift parameters, computed per patch via least squares [S1, unofficial].
- Patch size/overlap: Not given in the sources [G].

**Edge Cases and Pitfalls:**
- **Global vs. Local:** Global scale/shift fitting favors dominant objects, causing ill-fitted depths in less dominant sections due to inconsistencies in predicted depth differences [S1, unofficial]. Patch-wise adjustment alleviates this [S1, unofficial].
- **Computational Bottleneck:** The patch-based approach helps overcome the computational bottleneck of full image rendering [S1, unofficial].
- **Gradient Flow:** The stop-gradient operator is essential to prevent gradient flow into the MDE network during NeRF updates [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the number of seen views, patch count, and MDE inference time. The patch-wise approach reduces the computational bottleneck compared to full-image global fitting [S1, unofficial].
- **Memory Complexity:** Requires storing depth maps for seen views and MDE outputs. No specific memory bounds given [S1, unofficial].
- **Strengths:** Improves local geometric accuracy by handling local depth inconsistencies; prevents dominant objects from skewing alignment [S1, unofficial].
- **Limitations:** Requires careful patch definition; relies on the quality of the pre-trained MDE network [S1, unofficial].
- **When to use:** When sparse input views lead to geometric artifacts and local depth consistency is critical [S1, unofficial].
- **When not to use:** When computational resources are extremely limited and patch-wise processing overhead is prohibitive [G].

## Background
Originated in the DaRF paper (2023) [S1, unofficial]. The method addresses the issue where global scale/shift fitting undermines local statistics in NeRF geometry reconstruction from sparse inputs [S1, unofficial]. It builds on the observation that MDE outputs are relative and need alignment with absolute NeRF depth [S1, unofficial].

## Sources
[S1] DaRF: Boosting Radiance Fields from Sparse Inputs with Monocular Depth Adaptation, https://doi.org/10.48550/arxiv.2305.19201, fetched 2026-10-29 (unofficial)
