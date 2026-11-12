# Simple-RF

Also called: Simple Radiance Fields

## Overview
Simple-RF is a regularization framework for sparse-input radiance field optimization that mitigates floaters and duplication artifacts by supervising a high-capability "main" model with depth estimates from lower-capability "augmented" models [S1]. It sets itself apart from sibling methods like [3D Gaussian Splatting](1795-3d-gaussian-splatting.md), which require separate initialization studies for sparse inputs, by being applicable to various explicit (TensoRF, iNGP, ZipNeRF) and implicit (NeRF) models without modifying the inference pipeline [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of sparse input views (images and camera poses); a main radiance field model $F$ (e.g., NeRF, TensoRF, ZipNeRF); an augmented radiance field model $F_{aug}$ with reduced capacity.
*   **Outputs:** A trained main radiance field model $F$ capable of novel view synthesis; depth maps estimated during training for supervision.

**Data Structures:**
The radiance field $F$ is composed of two sub-networks $F = F_1 \bullet F_2$ [S1]:
1.  **$F_1$ (Geometry/Feature Network):** Takes a 3D point $p_i$ and outputs a view-independent volume density $\tilde{\tau}_i$ (or $\sigma_i$) and a latent feature vector $h_i$.
2.  **$F_2$ (Color Network):** Takes the latent feature $h_i$ and viewing direction $v$ to output the view-dependent color $c_i$.

For the augmented model $F_{aug}$, the architecture is identical to $F$ but with specific components disabled or reduced to lower capacity [S1].

**Numbered Steps:**

1.  **Model Initialization:** Initialize the main model $F$ and the augmented model $F_{aug}$ with reduced capacity. The reduction strategy depends on the base architecture (see Subtopics) [S1].
2.  **Ray Sampling:** For each training pixel, shoot a ray into the scene and sample $N$ 3D points $p_1, p_2, \dots, p_N$, where $p_1$ is closest to the camera and $p_N$ is farthest [S1].
3.  **Forward Pass (Main Model):** Query $F$ at each point $p_i$ to obtain density $\tilde{\tau}_i$ and latent feature $h_i$:
    $$ \tilde{\tau}_i, h_i = F_1 (p_i) $$
    $$ c_i = F_2 (h_i, v) $$
    where $v$ is the viewing direction [S1].
4.  **Forward Pass (Augmented Model):** Query $F_{aug}$ at each point $p_i$ to obtain density $\tilde{\tau}^{aug}_i$ and latent feature $h^{aug}_i$:
    $$ \tilde{\tau}^{aug}_i, h^{aug}_i = F_{aug,1} (p_i) $$
    $$ c^{aug}_i = F_{aug,2} (h^{aug}_i, v) $$
5.  **Volume Rendering:** Compute the rendered color $c$ for the pixel using the main model's samples:
    $$ c = \sum_{i=1}^{N} w_i c_i $$
    Compute the weights $w_i$ using the accumulated transmittance:
    $$ w_i = \exp \left( -\sum_{j=1}^{i-1} \delta_j \tilde{\tau}_j \right) \cdot (1 - \exp (-\delta_i \tilde{\tau}_i)) $$
    where $\delta_i$ is the distance between $p_i$ and $p_{i+1}$ [S1].
    Compute the expected depth $z$ for the main model:
    $$ z = \sum_{i=1}^{N} w_i z_i $$
    where $z_i$ is the depth of $p_i$ [S1].
    Similarly, compute the rendered color $c^{aug}$ and expected depth $z^{aug}$ for the augmented model using its samples [S1].
6.  **Depth Reliability Check:** Gauge the reliability of the depths estimated by the models by reprojecting pixels.
    *   Select a different nearest training view.
    *   Reproject the pixel coordinates from the current view to the nearest training view using the estimated depth $z$ (or $z^{aug}$).
    *   Compare the reprojected pixel color with the corresponding image in the nearest training view to determine the reprojection error [S1].
7.  **Loss Calculation:**
    *   Compute the photometric loss for the main model: $L_{ph} = \| c - \hat{c} \|^2$, where $\hat{c}$ is the true color of the pixel [S1].
    *   Determine which model provides more accurate depth based on the reprojection error.
    *   Apply a depth supervision loss $L_{aug}$ from the more reliable model to the other model. If the augmented model is more reliable, supervise the main model. If the main model is more reliable, supervise the augmented model (or skip if the main is already optimal, though the text implies bidirectional or selective supervision based on reliability) [S1].
    *   The total loss for the main model typically includes the photometric loss and the depth supervision loss if applicable [S1].
8.  **Backpropagation and Update:** Backpropagate the total loss through both models and update their parameters using an optimizer (e.g., Adam) [S1].
9.  **Inference:** Discard the augmented models. Use only the trained main model $F$ for novel view synthesis [S1].

**Parameters:**
*   $N$: The number of 3D points sampled along a ray for volume rendering. Typical values are not explicitly given in the sources, but standard NeRF uses 48-192 [G].
*   $c_i$: The view-dependent color at point $p_i$.
*   $v$: The viewing direction used to compute color from the latent feature.
*   $\delta_i$: The distance between adjacent sample points $p_i$ and $p_{i+1}$.

**Edge Cases and Pitfalls:**
*   **Inference Overhead:** Augmented models are not used during inference, ensuring no performance overhead for novel view synthesis [S1].
*   **Sub-optimal Solutions:** Naïvely reducing the capacity of radiance fields may lead to sub-optimal solutions in certain regions; hence, the main model is retained for final synthesis [S1].
*   **Depth Discontinuities:** Augmented models that learn only smooth depth transitions may fail to learn sharp depth discontinuities at object boundaries. The reliability check ensures that such inaccurate depths are not used for supervision [S1].
*   **3DGS Limitations:** The method is not directly applicable to 3D Gaussian Splatting (3DGS) without separate study, as 3DGS suffers from poor initialization with few input views, requiring a combination of good initialization and supervision [S1].

## Subtopics

### [NeRF with Softplus Activation and Importance Sampling](3524-nerf-with-softplus-activation-and-importance-sampling.md)
This method adapts the original NeRF architecture to enable stable optimization via a reparameterized volume sampling technique. It replaces the standard ReLU density activation with Softplus to ensure non-zero gradients and employs importance-sampling reparameterization for color estimation. It differs from full NeRF by using uniform weights for samples rather than a second "fine" network for re-weighting.

### [Gradient Scaling for Near-Camera Training](7379-gradient-scaling-for-near-camera-training.md)
Gradient scaling is a technique for differentiable volume rendering optimization that prevents background collapse and near-camera floaters by counter-balancing the sampling density imbalance close to cameras. It sets itself apart from alternatives like near-plane clipping or occlusion losses by addressing the root cause (gradient magnitude imbalance) directly during backpropagation, rather than imposing priors on density distribution or restricting ray origins.

### PlaNeRF
PlaNeRF is an unsupervised 3D plane regularization method for Neural Radiance Fields (NeRF) designed to improve large-scale urban scene reconstruction [S1]. It addresses the geometric instability of NeRF in low-texture, flat-surface environments by appending two regularization terms to the standard L2 photometric loss: a patch-based SSIM term for geometry initialization and an SVD-based plane regularization term for enforcing planar geometry [S1]. Unlike sibling methods such as [RegNeRF](link-to-regnerf) which enforce a plane orthogonal to the center ray, PlaNeRF targets the least-squares plane approximation of a patch, allowing the surface normal to converge in an unsupervised manner [S1].

### [Hard Sample Mining for NeRF Optimization](7722-hard-sample-mining-for-nerf-optimization.md)
Hard Sample Mining for NeRF Optimization is a training acceleration technique that reduces memory and compute costs by identifying and backpropagating gradients only for "hard" samples (those contributing significantly to the loss), while skipping the expensive backward pass for "easy" samples. It sets itself apart from sibling methods like [NeRF with Softplus Activation and Importance Sampling](3524-nerf-with-softplus-activation-and-importance-sampling.md) and [PlaNeRF](7394-planerf.md) by pruning samples based on their gradient contribution to the pixel loss rather than density or occupancy weights, and by using a two-forward-one-backward pass structure to bypass standard deep learning framework overheads associated with masked gradients.

### GeCoNeRF
GeCoNeRF is a regularization framework for few-shot Neural Radiance Field (NeRF) optimization that enforces geometric consistency across views using depth-guided warping and feature-level consistency modeling to overcome under-constrained optimization and overfitting [S1]. It sets itself apart from pixel-level photometric consistency methods (e.g., NeuralWarp, StructNeRF) by operating at the feature level, which preserves view-dependent specular effects that pixel-level losses would suppress [S1]. Unlike pre-training-based few-shot methods (e.g., PixelNeRF, MVSNeRF), it optimizes NeRF from scratch without requiring extensive multi-scene dataset priors [S1].

### Lambertian Augmentation
Lambertian Augmentation is an augmentation strategy for Simple-NeRF that disables view-dependent radiance in the coarse NeRF to mitigate shape-radiance ambiguity and duplication artifacts caused by the NeRF varying color based on viewing direction [S1]. It forces the model to output color based on the 3D point $p_i$ alone, as:
$$ \sigma_i, h_i = N_{1l} (\gamma (p_i, 0, l_p)); $$
$$ c_i = N_{2l} (h_i), $$
where $N_{1l}$ and $N_{2l}$ are the geometry and color networks of the Lambertian augmented model, and $\gamma$ is the positional encoding function [S1]. While the augmented model is more accurate in Lambertian regions, the main model remains better equipped to handle specular objects [S1].

### SinNeRF Loss and Annealing Strategy
This variant applies a specific loss annealing strategy to the Simple-RF framework. The overall loss function is:
$$ \mathcal{L}_{\text{total}} = \mathcal{L}_{\text{pix}} + \lambda_1 \mathcal{L}_{\text{geo}} + \lambda_2 \mathcal{L}_{\text{adv}} + \lambda_3 \mathcal{L}_{\text{cls}} $$
where $\lambda_1, \lambda_2, \lambda_3$ are weighting factors [S2]. The weights are annealed during training: initially, $\lambda_1 = 8$, $\lambda_2 = 0.1$, and $\lambda_3 = 0$. As training proceeds, $\lambda_2$ is gradually decreased to 0 and $\lambda_3$ is increased to 0.1 with a linear function [S2]. In early stages with large stride, the global structure prior ($\lambda_3$) has a large weight compared to local texture guidance ($\lambda_2$). As training proceeds and stride reduces, $\lambda_3$ is reduced and $\lambda_2$ is increased to focus on high-frequency details [S2].

## Operating characteristics
*   **Time Complexity:** Training time increases due to the simultaneous training of the augmented model alongside the main model. However, inference time is identical to the main model alone, as augmented models are discarded [S1].
*   **Memory Complexity:** Memory usage during training increases by the size of the augmented model's parameters. Inference memory usage is unchanged [S1].
*   **Measured Performance:** Simple-RF achieves significant improvement in performance on datasets including forward-facing scenes (NeRF-LLFF), unbounded forward-facing scenes (RealEstate-10K), unbounded 360° scenes (MipNeRF360), and bounded 360° scenes (NeRF-Synthetic) [S1].
*   **Scaling:** The method is applicable to various explicit models (TensoRF, iNGP, ZipNeRF) and implicit models (NeRF) [S1].
*   **Strengths:** Mitigates floaters and duplication artifacts in sparse input settings; no inference overhead; applicable to multiple radiance field architectures.
*   **Limitations:** Requires careful design of augmented models for each architecture; not directly applicable to 3DGS without separate initialization studies; relies on reprojection error for depth reliability which may be noisy in textureless regions.

## Background
Simple-RF was introduced in the paper "Simple-RF: Regularizing Sparse Input Radiance Fields with Simpler Solutions" by Nagabhushan Somraj, Sai Harsha Mupparaju, Adithyan Karanayil, and Rajiv Soundararajan [S1]. The work follows Occam’s razor principle, regularizing radiance fields to choose simpler solutions over complex ones, particularly in the few-shot setting [S1]. It addresses the shortcomings of existing methods like 3DGS which require separate initialization studies for sparse inputs [S1].

## Sources
[S1] Simple-RF: Regularizing Sparse Input Radiance Fields with Simpler Solutions, https://arxiv.org/abs/2404.19015v5, fetched 2026-10-27
[S2] SinNeRF: Training Neural Radiance Fields on Complex Scenes from a Single Image, https://doi.org/10.1007/978-3-031-20047-2_42, fetched 2026-10-26
