# Depth and Geometry Priors

Also called: Depth-supervised NeRF, RGB-D NeRF, Geometry-constrained NeRF

## Overview
Depth and Geometry Priors are optimization techniques that incorporate explicit depth information (from sensors, stereo, or monocular estimation) into the training of neural radiance fields to resolve the inherent scale ambiguity and geometric inaccuracies of standard volume rendering. They differ from sibling methods like [Relighting and Shading](17925-relighting-and-shading.md) or [Regularization and Constraints](17872-regularization-and-constraints.md) by directly supervising the geometric output (depth) rather than just the photometric output (color), thereby enforcing metric consistency and improving reconstruction speed and accuracy in sparse-view or ambiguous scenarios.

## Shared principles
The core mechanism involves augmenting the standard NeRF loss function with a geometric term that penalizes the discrepancy between the depth rendered by the neural field and a provided depth prior.

1.  **Depth Rendering**: The expected depth $\hat{D}(r)$ along a ray $r$ is computed as the weighted average of sample depths $t_i$, analogous to color rendering:
    $$
    \hat{D}(r) = \sum_{i=1}^N w_i t_i
    $$
    where $w_i = \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j)$ is the weight of the $i$-th sample, $\alpha_i$ is the opacity, and $t_i$ is the distance along the ray [S1].

2.  **Geometric Loss**: A loss term $l_g$ is added to the photometric loss $l_p$. The most common formulation is the L1 norm between the rendered depth $\hat{D}(r)$ and the ground truth or estimated depth $D(r)$ [S1][S2, unofficial]:
    $$
    l_g = \sum_{r \in R} |\hat{D}(r) - D(r)|
    $$
    In methods accounting for uncertainty, this is normalized by the depth variance $\sigma_D(r)$ to discourage weights on uncertain measurements [S1]:
    $$
    l_g = \sum_{r \in R} \frac{|\hat{D}(r) - D(r)|}{\sigma_D(r)}
    $$

3.  **Sample Guidance**: Depth priors are often used to guide the sampling strategy. Instead of uniform or stratified sampling along the entire ray, samples are concentrated around the expected depth value provided by the prior, reducing the number of required samples and accelerating convergence [S2, unofficial].

## Subtopics

### [Ambiguity-Aware Depth Estimates](16981-ambiguity-aware-depth-estimates.md)
Ambiguity-Aware Depth Estimates is a technique for learning a multimodal distribution of depth estimates conditioned on a single input image, using an implicit generative model to capture inherent ambiguities in monocular depth estimation such as non-opaque surfaces and albedo-shading confusion. Unlike GANs which optimize for similarity to a single ground truth, this method uses conditional Implicit Maximum Likelihood Estimation (cIMLE) to prevent mode collapse by enforcing that all ground truth points are explained by at least one generated sample. It is chosen when monocular depth estimation is ambiguous and a single deterministic estimate is insufficient. See [Ambiguity-Aware Depth Estimates](16981-ambiguity-aware-depth-estimates.md).

### [Mip-NeRF RGB-D](18129-mip-nerf-rgb-d.md)
Mip-NeRF RGB-D is a neural scene representation method that reconstructs 3D scenes from multi-view RGB-D data by combining the conical frustum sampling of Mip-NeRF with explicit depth supervision. It solves the limitations of standard NeRF regarding geometric accuracy and training speed by using depth information for local sampling and geometric loss, reducing training time by 3–5 times compared to standard NeRF. Unlike DS-NeRF which relies on sparse point clouds and reprojection errors, Mip-NeRF RGB-D uses dense RGB-D frames and explicitly models depth uncertainty to improve photometry and geometry. It is chosen when high-quality dense RGB-D data is available and fast, accurate geometry is required. See [Mip-NeRF RGB-D](18129-mip-nerf-rgb-d.md).

### SparseNeRF
SparseNeRF addresses the problem of synthesizing new views from finite perspectives by extracting depth priors from pre-trained depth models or rough depth maps captured by consumer-grade depth sensors [S1]. It introduces local depth ordering and spatial continuity constraints to extract robust depth priors from rough depth maps, thereby improving the performance of new view synthesis while maintaining geometric spatial continuity [S1]. It is chosen when only sparse or low-quality depth data is available from consumer sensors. Pitfall: Relies on pre-trained depth models or depth maps captured by consumer-grade sensors, the accuracy and quality of which can be limited by real-world conditions [S1].

### Depth-Guided NeRF Training
Depth-Guided NeRF Training uses a multistage pipeline for depth supervision to guide NeRF sample placement [S2, unofficial]. It applies a pretrained sparse-to-dense depth completion network to sparse structure-from-motion depth estimates to guide sample placement and supervise NeRF depth [S2, unofficial]. The steps involve finetuning a single-view depth estimation network on sparse multiview stereo depth estimates, using the resulting dense depth maps to guide NeRF sample placement, and supervising the depth produced by NeRF [S2, unofficial]. It is chosen when starting with sparse SfM depth estimates that need to be densified for effective NeRF training.

### NeRF with Depth Completion Guidance
NeRF with Depth Completion Guidance applies a pretrained sparse-to-dense depth completion network to sparse structure-from-motion depth estimates to guide sample placement and supervise NeRF depth [S2, unofficial]. It uses the resulting depth and predicted uncertainty to guide sample placement and supervise the depth produced by NeRF [S2, unofficial]. It is chosen when uncertainty-aware depth completion is needed to robustly handle sparse input data.

## Background
The integration of depth priors into NeRF originated from the observation that standard NeRF suffers from scale ambiguity and poor geometric accuracy, especially with sparse views. Early works like NerfingMVS [WLR* 21] and Roessle et al. [RBM* 21] introduced multistage pipelines using sparse-to-dense depth completion to guide sampling and supervise depth [S2, unofficial]. SparseNeRF [38] (2023) further advanced this by using local depth ordering and spatial continuity constraints from rough depth maps [S1]. Mip-NeRF RGB-D [S1] combined conical frustum sampling with explicit depth supervision to improve both geometry and training speed. These methods are widely used in applications requiring metric 3D reconstruction from limited or noisy depth data, such as autonomous driving, robotics, and cultural heritage digitization.

## Sources
[S1] Neural Radiance Field-based Visual Rendering: a Comprehensive Review, https://ieeexplore.ieee.org/abstract/document/11455380, fetched 2026-10-30
[S2] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
