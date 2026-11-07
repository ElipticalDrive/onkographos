# Regularization and Loss Functions

Also called: NeRF Regularization, Geometric Priors, View Synthesis Losses

## Overview
Regularization and Loss Functions address the severe under-constraint problem in Neural Radiance Fields (NeRFs), where limited or inconsistent input views lead to geometric artifacts (e.g., "floaters"), overfitting, and implausible geometry [S1, 7381-diffusion-regularized-nerf-training.md, 8582-regnerf.md]. This family sets itself apart from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on the objective function rather than the sampling strategy, and from [Mip-NeRF 360](3306-mip-nerf-360.md) by explicitly introducing priors (diffusion, flow, or geometric) to constrain the solution space beyond photometric consistency.

## Shared principles
All methods in this family share the fundamental structure of an augmented loss function. The total loss $\mathcal{L}_{\text{total}}$ is a weighted sum of the standard photometric reconstruction loss and one or more regularization terms $\mathcal{L}_k$:

$$ \mathcal{L}_{\text{total}} = \mathcal{L}_{\text{photo}} + \sum_k \lambda_k \mathcal{L}_k $$

where $\lambda_k$ are hyperparameters controlling the strength of each regularizer [S1, 7475-geometric-baseline-regularization.md, 8582-regnerf.md]. The photometric loss is typically the Mean Squared Error (MSE) between the rendered color $\mathbf{C}$ and the ground truth image $I$:

$$ \mathcal{L}_{\text{photo}} = || \mathbf{C} - I ||_2^2 $$

Regularization terms $\mathcal{L}_k$ operate on the density $\sigma$ and color $\mathbf{c}$ fields or the rendered weights $w_i$ to constrain the solution space [S1, 7475-geometric-baseline-regularization.md, 8582-regnerf.md]. Common strategies include enforcing spatial smoothness, ensuring geometric consistency across views, or aligning the output distribution with learned priors.

## Subtopics

### [Diffusion-Regularized NeRF Training](7381-diffusion-regularized-nerf-training.md)
Diffusion-Regularized NeRF Training addresses the severe under-constraint problem in Neural Radiance Fields (NeRFs), where limited input views lead to geometric artifacts such as "floaters" (semi-transparent material floating in mid-air) and implausible geometry. It sets itself apart from sibling methods like RegNeRF by directly regularizing geometry via a Denoising Diffusion Model (DDM) trained on RGBD (color and depth) patches, whereas RegNeRF uses normalizing flows on RGB patches and Dreamfusion uses DDMs for text-guided generation rather than image-based regularization. The method incorporates a learned prior over the joint distribution of color and depth, backpropagating the DDM's score function (gradient of the log-likelihood) to the NeRF's density and color fields during optimization.

### [Geometric Baseline Regularization](7475-geometric-baseline-regularization.md)
Geometric Baseline Regularization is a composite loss function used to stabilize the optimization of Neural Radiance Fields (NeRFs), particularly in few-view settings where standard photometric loss leads to degenerate solutions. It augments the standard photometric reconstruction loss with three geometric terms: foreground completeness, frustum consistency, and depth distribution compactness. Unlike sibling methods that rely on external priors (e.g., diffusion models [7381-diffusion-regularized-nerf-training.md]) or explicit depth supervision [8582-regnerf.md], this method enforces geometric plausibility directly through the volume rendering weights and ray sampling structure.

### [RegNeRF](8582-regnerf.md)
RegNeRF is a regularization framework for Neural Radiance Fields (NeRF) designed to improve view synthesis quality from sparse or inconsistent input views. It addresses the problem of MLP overfitting and unrealistic interpolation in few-shot scenarios by introducing priors that constrain unobserved views. Unlike its sibling [Geometric Baseline Regularization](7475-geometric-baseline-regularization.md), which often relies on explicit geometric constraints or depth maps, RegNeRF operates by augmenting the training loss with a Normalizing flow prior and a Smoothness prior.

### [Occlusion-Eliminating NeRF Variant](8653-occlusion-eliminating-nerf-variant.md)
This method addresses the degradation of novel view synthesis caused by foreground occlusions (e.g., people, vehicles) that obscure the target scene during training data capture. It solves this by decoupling the scene representation into two distinct neural networks: a scene MLP for the target geometry and a background MLP for the occluding elements. Unlike standard NeRF which treats all density as part of a single volume, this variant explicitly models the occluder to restore an occlusion-free background, distinguishing it from general regularization methods like RegNeRF [8582-regnerf.md] which do not explicitly separate occluders.

### [Depth and Geometry Supervision](8727-depth-and-geometry-supervision.md)
Depth and Geometry Supervision methods accelerate Neural Radiance Field (NeRF) optimization and improve geometric accuracy in sparse-view or single-view settings by incorporating external depth priors into the training loop. Unlike standard NeRF, which relies solely on photometric consistency and suffers from shape-radiance ambiguity, these methods constrain the density field $\sigma$ to align with estimated surface locations, reducing overfitting and speeding up convergence. They differ from sibling methods like [Diffusion-Regularized NeRF Training](7381-diffusion-regularized-nerf-training.md) by using explicit geometric targets (depth maps or keypoints) rather than implicit generative priors, and from [Geometric Baseline Regularization](7475-geometric-baseline-regularization.md) by directly supervising the ray termination distribution or sampling bounds rather than just density smoothness.

### AligNeRF Training Objective
AligNeRF replaces the standard per-pixel MSE loss used in mip-NeRF 360 with a combination of patch-based MSE (for misalignment) and shallow VGG feature-space loss (for high-frequency detail) [S1]. The total loss is defined as:
$$ L_{total} = L_{PM} + w \cdot L_{HF} $$
where $w$ is the weight for the high-frequency loss, typically set to 0.05 [S1].

### Coarse-Fine Consistency Loss
Coarse-Fine Consistency Loss enforces consistency between depth estimates of the coarse and fine NeRF models to prevent ineffective hierarchical sampling and blur [S2]. The loss is defined as:
$$ L_{cfc} = \mathbf{1}_{\{m_f=1\}} \odot ||z_c - \hat{\nabla}(z_f)||^2 + \mathbf{1}_{\{m_c=1\}} \odot ||\hat{\nabla}(z_c) - z_f||^2 $$
where $z_c$ and $z_f$ are coarse and fine depths, and $m_c/m_f$ are reliability masks determined by reprojection [S2].

### Cache Consistency Loss
Cache Consistency Loss supervises the physically-based model to constrain the appearance of the NeRF cache by matching diffuse and specular colors [S3]. It involves supervising cache diffuse color $c_{\text{cache}}(\mathbf{x})$ and specular color $c_{\text{cache}}(\mathbf{x}, \omega_i)$ to match physically-based model colors $c_{\text{phys}}(\mathbf{x})$ and $c_{\text{phys}}(\mathbf{x}, \omega_i)$. Additionally, it predicts an additional output from the cache $c_{\text{irradiance}}(\mathbf{x}, \omega_o)$ and supervises it to match irradiance from the physically-based model (computed by setting BRDF to perfectly Lambertian with $a(\mathbf{x}) = 1$) [S3].

## Background
Regularization in NeRFs originated from the observation that sparse input views lead to degenerate solutions, such as "floaters" and view-dependent artifacts [S1, 7381-diffusion-regularized-nerf-training.md, 8582-regnerf.md]. Early methods like RegNeRF introduced normalizing flows to constrain output distributions [S1]. Subsequent work, such as Geometric Baseline Regularization, focused on explicit geometric constraints like frustum consistency [S1]. More recent approaches leverage generative models, such as Diffusion-Regularized NeRF Training, to enforce realistic RGBD distributions [S1]. The field has also seen specialized losses for specific artifacts, such as occlusion elimination [8653-occlusion-eliminating-nerf-variant.md] and alignment-aware training for high-frequency details [S1].

## Sources
[S1] AligNeRF: High-Fidelity Neural Radiance Fields via Alignment-Aware Training, https://doi.org/10.1109/cvpr52729.2023.00013, fetched 2026-10-26
[S2] Simple-RF: Regularizing Sparse Input Radiance Fields with Simpler Solutions, https://arxiv.org/abs/2404.19015v5, fetched 2026-10-27
[S3] Flash Cache: Reducing Bias in Radiance Cache Based Inverse Rendering, https://doi.org/10.1007/978-3-031-73390-1_2, fetched 2026-10-27
