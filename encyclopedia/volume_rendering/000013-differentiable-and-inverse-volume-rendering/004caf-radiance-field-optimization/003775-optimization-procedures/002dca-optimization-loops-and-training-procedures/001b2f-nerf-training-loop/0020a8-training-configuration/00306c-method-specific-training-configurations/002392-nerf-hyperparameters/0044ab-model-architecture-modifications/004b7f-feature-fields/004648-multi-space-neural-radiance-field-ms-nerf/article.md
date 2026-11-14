# Multi-Space Neural Radiance Field (MS-NeRF)

Also called: MS-Mip-NeRF 360, MS-NeRFB, Multi-space NeRF

## Overview
MS-NeRF addresses the failure of standard NeRF methods to render reflective and refractive objects, which violate multi-view consistency in Euclidean space and result in blurry or distorted images [S1]. It differs from single-space NeRF by representing the scene using a group of feature fields in parallel sub-spaces rather than a single radiance field, allowing virtual images to be placed in sub-spaces visible only from specific viewpoints [S1]. This approach automatically handles mirror-like objects without manual labeling, unlike vanilla NeRF which treats virtual images as textures on reflective surfaces [S1].

## How it works
MS-NeRF operates as an enhancement module that replaces the original output layer of a backbone NeRF method (MLP-based or grid-based) with a multi-space module [S1]. It builds on the [Feature Fields](12995-feature-fields.md) principle where the network predicts arbitrary per-point feature signals in addition to color and density [S1, S2, S3, S4].

**Inputs and Outputs:**
*   **Input:** Ray $r$ defined by origin and direction; backbone features/densities from the underlying NeRF architecture.
*   **Output:** Final rendered color $\hat{C}(r)$ for the pixel.

**Data Structures and Parameters:**
*   $K$: Hyperparameter for the total number of sub-spaces [S1].
*   $d$: Hyperparameter for the feature dimension of the neural feature field [S1].
*   $\{\sigma_{ik}\}$: Density at sample point $i$ in sub-space $k$ [S1].
*   $\{f_{ik}\}$: Feature vector of dimension $d$ at sample point $i$ in sub-space $k$ [S1].
*   $\Theta_D$: Decoder MLP with one hidden layer [S1].
*   $\Theta_G$: Gate MLP with one hidden layer [S1].

**Numbered Steps:**

1.  **Multi-Head Output:** Replace the backbone's final output layer with a multi-head layer. For each sample point $i$ along the ray, the network outputs $K$ densities $\{\sigma_{ik}\}$ and $K$ feature vectors $\{f_{ik}\}$ of dimension $d$, where each pair corresponds to a specific sub-space $k \in \{1, \dots, K\}$ [S1].

2.  **Per-Subspace Volume Rendering:** Integrate features along the ray in each sub-space independently to collect $K$ feature maps. For a ray $r$, the rendered feature $\hat{F}^k(r)$ for sub-space $k$ is calculated as [S1]:
    $$
    \hat{F}^k(r) = \sum_{i=1}^{N} T_{ik} (1 - \exp(-\sigma_{ik} \delta_i)) f_{ik}
    $$
    where $T_{ik} = \exp(-\sum_{j=1}^{i-1} \sigma_{jk} \delta_j)$ is the accumulated transmittance for sub-space $k$, and $\delta_i$ is the distance between sample points [S1].

3.  **Decoder and Gate Processing:** The aggregated features $\{\hat{F}^k\}$ for all $K$ sub-spaces are passed through two separate MLPs [S1]:
    *   The Decoder MLP $\Theta_D$ takes $\{\hat{F}^k\}$ as input and outputs RGB color vectors $\{C_k\}$ for each sub-space:
        $$
        \Theta_D\{\hat{F}^k\} \rightarrow \{C_k\}
        $$
    *   The Gate MLP $\Theta_G$ takes $\{\hat{F}^k\}$ as input and outputs scalar weights $\{w_k\}$ that control the visibility of each sub-space:
        $$
        \Theta_G\{\hat{F}^k\} \rightarrow \{w_k\}
        $$
    Both $\Theta_D$ and $\Theta_G$ are MLPs with one hidden layer [S1].

4.  **Softmax Weighting:** Apply the softmax function to the gate weights $\{w_k\}$ to determine the contribution of each sub-space to the final color. The final rendered color $\hat{C}(r)$ is computed as [S1]:
    $$
    \hat{C}(r) = \sum_{k=1}^{K} \frac{\exp(w_k)}{\sum_{i=1}^{K} \exp(w_i)} C_k
    $$

**Edge Cases and Pitfalls:**
*   The method requires no additional loss terms compared to vanilla NeRF, relying on standard photometric loss [S1].
*   Computational overhead is small for training and inference [S1].
*   The composition weights of sub-spaces change dynamically according to spatial location and view direction, allowing the model to handle complex light paths [S1].

## Operating characteristics
*   **Performance:** On the Mip-NeRF 360 dataset, MS-NeRF improves PSNR by 4.15 dB on reflective regions with only 0.5% extra parameters, without degrading rendering quality on other regions [S1]. On TensoRF, it improves PSNR by 2.71 dB on reflective regions with 0.046% extra parameters [S1].
*   **Complexity:** The method introduces small computational overheads for training and inferring the extra-space outputs [S1].
*   **Strengths:** Automatically handles mirror-like objects in 360-degree rendering without manual labeling; overcomes multi-view consistency violations by placing virtual images in specific sub-spaces; works as a general enhancement for both MLP-based (NeRF, Mip-NeRF, Mip-NeRF 360) and grid-based (TensoRF, iNGP) methods [S1].
*   **Limitations:** Performance on non-reflective regions is not degraded but not improved beyond the backbone; requires careful tuning of $K$ and $d$ [S1].

## Background
MS-NeRF was proposed to address the limitations of existing NeRF methods in handling reflective and refractive objects, which often result in blurry or distorted rendering due to violated multi-view consistency in Euclidean space [S1]. Existing datasets like RFFR and Shiny lacked sufficient coverage of 360-degree rendering with mirror-like objects, prompting the construction of a novel dataset with 33 synthetic and 7 real captured scenes [S1]. The method was published in IEEE Transactions on Pattern Analysis and Machine Intelligence in 2025, with a preliminary version appearing at CVPR [S1].

## Sources
[S1] MS-NeRF: Multi-Space Neural Radiance Fields, https://doi.org/10.1109/tpami.2025.3540074, fetched 2026-10-26
