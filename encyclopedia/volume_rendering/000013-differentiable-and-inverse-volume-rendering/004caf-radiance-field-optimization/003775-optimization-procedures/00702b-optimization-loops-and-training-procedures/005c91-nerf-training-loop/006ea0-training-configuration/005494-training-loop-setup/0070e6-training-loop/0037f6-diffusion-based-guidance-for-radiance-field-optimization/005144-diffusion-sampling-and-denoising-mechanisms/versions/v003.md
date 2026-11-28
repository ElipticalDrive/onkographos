# Diffusion Sampling and Denoising Mechanisms
Also called: Diffusion Reverse Process, Score-Based Sampling

## Overview
This family defines the algorithms and network architectures used to invert the forward diffusion process, recovering clean latent representations of radiance fields from noisy inputs. It solves the problem of generating or reconstructing 3D scenes by iteratively denoising a latent volume or code. It differs from direct optimization methods by leveraging the learned data prior of a diffusion model to guide the reconstruction, often using classifier-free guidance to steer the output toward conditional targets .

## Shared principles
All methods in this family operate on a latent space $x$ (e.g., voxel grids, triplanes, or scene codes) and a noise schedule defined by timesteps $t \in \{1, \dots, T\}$. The forward process adds Gaussian noise to a clean sample $x_0$ to produce $x_t$:
$$ x_t = \sqrt{\bar{\alpha}_t} x_0 + \sqrt{1 - \bar{\alpha}_t} \epsilon, \quad \epsilon \sim \mathcal{N}(0, I) $$
where $\bar{\alpha}_t = \prod_{i=1}^t (1 - \beta_i)$ is the cumulative product of variance schedule coefficients $\beta_i$ .

The core mechanism is the prediction of the noise component $\epsilon_\theta(x_t, t)$ or the velocity $v_\theta(x_t, t)$ using a neural network. The denoised estimate $\hat{x}_0$ is recovered via:
$$ \hat{x}_0 = \frac{x_t - \sqrt{1 - \bar{\alpha}_t} \epsilon_\theta(x_t, t)}{\sqrt{\bar{\alpha}_t}} $$
for noise prediction parameterization .

Sampling proceeds from $t=T$ down to $t=0$. The transition $x_t \to x_{t-1}$ generally follows the form:
$$ x_{t-1} = \mu_\theta(x_t, t) + \sigma_t \epsilon $$
where $\mu_\theta$ is the mean prediction and $\sigma_t$ controls stochasticity. Deterministic samplers (e.g., DDIM) set $\sigma_t = 0$, allowing exact reconstruction from noise in the limit of infinite steps . Stochastic samplers (e.g., DDPM) retain noise to preserve diversity.

Classifier-Free Guidance (CFG) is a shared steering mechanism. Given conditional $c$ and unconditional $c_{uncond}$ inputs, the predicted noise is combined as:
$$ \epsilon_t = \epsilon_\theta(x_t, t, c_{uncond}) + s \cdot (\epsilon_\theta(x_t, t, c) - \epsilon_\theta(x_t, t, c_{uncond})) $$
where $s$ is the guidance scale .

## Subtopics

### [DDIM Sampling with Classifier-Free Guidance](8118-ddim-sampling-with-classifier-free-guidance.md)
DDIM Sampling with Classifier-Free Guidance is a deterministic (or stochastic) reverse diffusion algorithm that generates or refines samples in fewer steps than standard DDPM by skipping intermediate noise timesteps, while applying CFG to steer the generation towards conditional targets. It builds on [Diffusion Sampling and Denoising Mechanisms](14430-diffusion-sampling-and-denoising-mechanisms.md) by providing a specific discrete-time integration scheme for the reverse process and a mechanism to combine conditional and unconditional model predictions. Unlike [DDPM Denoise Function Network](11102-ddpm-denoise-function-network.md), which relies on the stochastic posterior sampling of DDPM, DDIM allows for exact reconstruction of the clean sample from noise in the limit of infinite steps and supports deterministic trajectories.

### [DDPM Denoise Function Network](11102-ddpm-denoise-function-network.md)
The DDPM Denoise Function Network is a neural network architecture that parameterizes the reverse diffusion process for 3D radiance fields, predicting the noise component $\epsilon_\theta(x_t, t)$ to recover clean latent volumes. It enables image-to-volume synthesis by integrating volumetric rendering gradients into the diffusion sampling loop, steering the denoised field toward photometric consistency with target images. It differs from sibling methods by explicitly using a 3D convolutional U-Net structure to operate directly on voxelized latent representations rather than implicit coordinate-based MLPs.

### Image-Guided Sampling
#### definition
- [S1] A sampling method that guides the denoising process of a diffusion model using gradients of the rendering loss against known observations to reconstruct 3D scenes from images.
#### step
- [S1] Combine guidance gradients $g$ with unconditional score prediction as a correction to the denoising output $\hat{x}$: $\hat{x} \leftarrow \hat{x} - \lambda_{gd}(t) g$.
- [S1] Use a predictor-corrector sampler to solve for $x(0)$ by alternating between a DDIM step and multiple Langevin correction steps.
#### parameter
- [S1] $\omega$: Hyperparameter for SNR-based weighting, chosen as 0.5 or 0.25.
- [S1] $\lambda_{gd}$: Guidance scale used to combine guidance gradients with unconditional score prediction.

## Background
Diffusion models were originally introduced for image generation [S1]. In the context of 3D radiance fields, they were adapted to learn priors over scene representations (such as triplanes or voxel codes) to enable single-stage generation and reconstruction [S1]. The integration of diffusion sampling with differentiable rendering allows for test-time reconstruction from images by guiding the latent denoising process with rendering gradients [S1]. This approach contrasts with two-stage methods that train autoencoders separately from the diffusion prior [S1].

## Sources
[S1] Single-Stage Diffusion NeRF: A Unified Approach to 3D Generation and Reconstruction, https://doi.org/10.1109/iccv51070.2023.00229, fetched 2026-10-29
