# Regularization and Loss Functions
## Overview
This family encompasses the loss functions and regularization terms added to the standard photometric reconstruction loss to stabilize the optimization of Neural Radiance Fields (NeRFs), particularly in few-view, sparse, or ill-posed settings [S1, 7475-geometric-baseline-regularization.md, 8582-regnerf.md]. While standard photometric loss minimizes pixel-wise differences between rendered and ground truth images, it suffers from shape-radiance ambiguity and overfitting to noise [S1, 8727-depth-and-geometry-supervision.md]. This family addresses these issues by introducing geometric priors, distributional constraints, and multi-GPU consistency terms. It sets itself apart from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) (which focuses on sampling strategies) and [NeRF Training Loop](970-nerf-training-loop.md) (which focuses on the optimization schedule) by specifically defining the mathematical penalties applied to the model parameters during gradient descent.
## Shared principles
All methods in this family share the fundamental structure of an augmented loss function. The total loss $\mathcal{L}_{\text{total}}$ is a weighted sum of the standard photometric reconstruction loss and one or more regularization terms $\mathcal{L}_k$:

$$ \mathcal{L}_{\text{total}} = \mathcal{L}_{\text{photo}} + \sum_k \lambda_k \mathcal{L}_k $$

where $\lambda_k$ are hyperparameters controlling the strength of each regularizer [S1, 7475-geometric-baseline-regularization.md, 8582-regnerf.md]. The photometric loss is typically the Mean Squared Error (MSE) between the rendered color $\mathbf{C}$ and the ground truth image $I$:

$$ \mathcal{L}_{\text{photo}} = || \mathbf{C} - I ||_2^2 $$

Regularization terms $\mathcal{L}_k$ operate on the density $\sigma$ and color $\mathbf{c}$ fields or the rendered weights $w_i$ to constrain the solution space [S1, 7475-geometric-baseline-regularization.md, 8582-regnerf.md]. Common strategies include enforcing spatial smoothness, ensuring geometric consistency across views, or aligning the output distribution with learned priors.
## Subtopics

### [Diffusion-Regularized NeRF Training](7381-diffusion-regularized-nerf-training.md)
Addresses severe under-constraint problems in sparse views by regularizing geometry via a Denoising Diffusion Model (DDM) trained on RGBD patches. It differs from [RegNeRF](8582-regnerf.md) by using a DDM score function rather than normalizing flows, and from [Geometric Baseline Regularization](7475-geometric-baseline-regularization.md) by relying on generative priors rather than explicit geometric constraints. Choose this when dealing with "floaters" and implausible geometry in very sparse settings.

### [Geometric Baseline Regularization](7475-geometric-baseline-regularization.md)
A composite loss function stabilizing optimization in few-view settings by augmenting photometric loss with foreground completeness, frustum consistency, and depth distribution compactness. Unlike [Diffusion-Regularized NeRF Training](7381-diffusion-regularized-nerf-training.md), it does not use external generative models. Choose this for a lightweight, explicit geometric prior that prevents degenerate solutions without the complexity of diffusion models.

### [RegNeRF](8582-regnerf.md)
Improves view synthesis from sparse/inconsistent views by adding a Normalizing Flow prior and a Smoothness prior to the loss. It differs from [Geometric Baseline Regularization](7475-geometric-baseline-regularization.md) by operating on output distributions and spatial gradients rather than explicit depth/weight constraints. Choose this when MLP overfitting is the primary issue and a statistical prior on density/color is preferred over geometric constraints.

### [Occlusion-Eliminating NeRF Variant](8653-occlusion-eliminating-nerf-variant.md)
Decouples scene representation into a Scene MLP and a Background MLP to explicitly model and remove foreground occlusions. Unlike general regularizers like [RegNeRF](8582-regnerf.md), it solves occlusion by architectural separation and specific depth inconsistency priors on the background model. Choose this when training data contains significant foreground occluders (e.g., people) obscuring the target scene.

### [Depth and Geometry Supervision](8727-depth-and-geometry-supervision.md)
Accelerates optimization and improves geometric accuracy by incorporating external depth estimates or structural constraints. Distinguished from [Diffusion-Regularized NeRF Training](7381-diffusion-regularized-nerf-training.md) by using explicit depth targets rather than learned priors, and from [Geometric Baseline Regularization](7475-geometric-baseline-regularization.md) by focusing on alignment with specific depth maps. Choose this when reliable depth estimates (from MVS or sensors) are available.

### [NeRF-XL Multi-GPU Distortion Loss](10377-nerf-xl-multi-gpu-distortion-loss.md)
A distributed regularization term enforcing geometric consistency in large-scale NeRF training by penalizing weight ambiguity along rays across multiple GPUs. Unlike single-GPU distortion losses, it aggregates local density/transmittance statistics from all participating GPUs. Choose this for large-scale distributed training where ray segments are partitioned across hardware.

#### AligNeRF Training Objective
Replaces standard per-pixel MSE with a combination of patch-based MSE (for misalignment) and shallow VGG feature-space loss (for high-frequency detail) [S1]. Formula: $L_{total} = L_{PM} + w \cdot L_{HF}$ [S1]. Parameter $w$ is typically 0.05 [S1].

#### Coarse-Fine Consistency Loss
Enforces consistency between depth estimates of coarse and fine NeRF models to prevent blur and ineffective hierarchical sampling [S2]. Formula: $L_{cfc} = \mathbf{1}_{\{m_f=1\}} \odot ||z_c - \hat{\nabla}(z_f)||^2 + \mathbf{1}_{\{m_c=1\}} \odot ||\hat{\nabla}(z_c) - z_f||^2$, where $z_c, z_f$ are depths and $m_c, m_f$ are reliability masks [S2].

#### Cache Consistency Loss
Supervises a physically-based model to constrain the appearance of a NeRF cache by matching diffuse and specular colors [S3]. Steps: 1) Supervise cache diffuse color $c_{\text{cache}}(\mathbf{x})$ and specular color $c_{\text{cache}}(\mathbf{x}, \omega_i)$ to match physically-based model colors $c_{\text{phys}}(\mathbf{x})$ and $c_{\text{phys}}(\mathbf{x}, \omega_i)$. 2) Predict additional cache output $c_{\text{irradiance}}(\mathbf{x}, \omega_o)$ and supervise it to match irradiance from the physically-based model (computed by setting BRDF to perfectly Lambertian with $a(\mathbf{x}) = 1$) [S3].

## Background
The problem of regularization in NeRF optimization emerged alongside the original NeRF paper [Kerbl et al., 2023] and subsequent works addressing sparse view synthesis. RegNeRF [Mildenhall et al., 2021] introduced normalizing flows and smoothness priors to handle sparse inputs. Geometric Baseline Regularization and Diffusion-Regularized NeRF Training [S1] represent more recent efforts to explicitly constrain geometry using diffusion models and explicit geometric terms, respectively, to combat floaters and ambiguity. Multi-GPU distortion loss [S1] addresses scalability issues in large-scale training.
## Sources
[S1] AligNeRF: High-Fidelity Neural Radiance Fields via Alignment-Aware Training, https://doi.org/10.1109/cvpr52729.2023.00013, fetched 2026-10-26
[S2] Simple-RF: Regularizing Sparse Input Radiance Fields with Simpler Solutions, https://arxiv.org/abs/2404.19015v5, fetched 2026-10-27
[S3] Flash Cache: Reducing Bias in Radiance Cache Based Inverse Rendering, https://doi.org/10.1007/978-3-031-73390-1_2, fetched 2026-10-27
