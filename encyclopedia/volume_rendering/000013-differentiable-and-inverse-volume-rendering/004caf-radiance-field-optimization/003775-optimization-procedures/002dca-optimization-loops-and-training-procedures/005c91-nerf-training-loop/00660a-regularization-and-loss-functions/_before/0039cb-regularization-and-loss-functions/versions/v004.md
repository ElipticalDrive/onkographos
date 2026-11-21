# Regularization and Loss Functions
Also called: NeRF Regularization, Geometric Priors, Loss Augmentation

## Overview
This family encompasses loss functions and regularization techniques applied during the optimization of Neural Radiance Fields (NeRFs) to mitigate the ill-posed nature of inverse volume rendering, particularly in sparse-view or unbounded settings. While standard NeRF training relies solely on photometric consistency ($\mathcal{L}_{\text{photo}}$), these methods augment the objective with geometric, statistical, or diffusion-based priors to suppress artifacts such as "floaters," overfitting, and view-dependent hallucinations. They set themselves apart from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) (which focuses on sampling efficiency) and [Training Configuration](2332-training-configuration.md) (which focuses on hyperparameter schedules) by directly modifying the gradient signal to enforce structural plausibility.

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
Depth and Geometry Supervision accelerates Neural Radiance Field (NeRF) optimization and improves geometric accuracy by incorporating explicit depth constraints into the training loss, addressing the shape-radiance ambiguity inherent in volume rendering. Unlike standard NeRF, which relies solely on photometric consistency, this family of methods enforces that the ray termination distribution or sampling bounds align with external depth priors, such as Structure-from-Motion (SfM) keypoints, monocular estimates, or multi-view consistent depth maps.

### AligNeRF Training Objective
AligNeRF replaces the standard per-pixel MSE loss used in mip-NeRF 360 with a combination of patch-based MSE (for misalignment) and shallow VGG feature-space loss (for high-frequency detail) [S1]. The total loss is defined as:
$$ L_{total} = L_{PM} + w \cdot L_{HF} $$
where $L_{PM}$ is the patch-based MSE and $L_{HF}$ is the high-frequency loss derived from the first block of a pre-trained VGG-19 model [S1]. The weight $w$ is empirically set to 0.05 [S1]. This approach improves high-frequency details while preserving real textures, addressing the issue where misalignment prevents the recovery of high-frequency detail [S1].

### Coarse-Fine Consistency Loss
Coarse-Fine Consistency Loss is a loss function that enforces consistency between depth estimates of the coarse and fine NeRF models to prevent ineffective hierarchical sampling and blur [S2]. It is defined as:
$$ L_{cfc} = \mathbb{1}_{\{m_f=1\}} \odot ||z_c - \hat{\nabla}(z_f)||^2 + \mathbb{1}_{\{m_c=1\}} \odot ||\hat{\nabla}(z_c) - z_f||^2 $$
where $z_c$ and $z_f$ are coarse and fine depths, and $m_c/m_f$ are reliability masks determined by reprojection [S2]. This loss drives the two NeRFs to be consistent in their solutions, preventing blur in rendered images [S2].

## Background
The problem of overfitting and geometric artifacts in NeRFs, particularly with sparse inputs, was identified early in the field. RegNeRF [8582-regnerf.md] introduced normalizing flows and smoothness priors to constrain unobserved views. Geometric Baseline Regularization [7475-geometric-baseline-regularization.md] provided a lightweight alternative using frustum consistency and depth compactness. More recently, diffusion models have been adapted for regularization, with Diffusion-Regularized NeRF Training [7381-diffusion-regularized-nerf-training.md] leveraging DDMs trained on synthetic RGBD data to guide optimization. AligNeRF [S1] addressed misalignment artifacts in unbounded scenes by modifying the loss to include high-frequency features. Simple-RF [S2] introduced coarse-fine consistency to stabilize hierarchical sampling. These methods are widely used in novel view synthesis, particularly for sparse-view reconstruction, dynamic scenes, and unbounded outdoor environments.

## Sources
[S1] AligNeRF: High-Fidelity Neural Radiance Fields via Alignment-Aware Training, https://doi.org/10.1109/cvpr52729.2023.00013, fetched 2026-10-26
[S2] Simple-RF: Regularizing Sparse Input Radiance Fields with Simpler Solutions, https://arxiv.org/abs/2404.19015v5, fetched 2026-10-27
