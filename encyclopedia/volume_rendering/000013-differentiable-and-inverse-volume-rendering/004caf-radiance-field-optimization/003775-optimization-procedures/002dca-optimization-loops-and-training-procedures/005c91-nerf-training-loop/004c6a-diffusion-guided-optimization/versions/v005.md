# Diffusion-guided Optimization


## Overview
Diffusion-guided optimization methods train Neural Radiance Fields (NeRFs) by leveraging pretrained 2D or 3D diffusion models as priors to guide the optimization of 3D geometry and texture [G]. These methods solve the problem of 3D data scarcity and underconstrained reconstruction by distilling generative knowledge from 2D image distributions into 3D scene representations [G]. They set themselves apart from standard NeRF training by backpropagating gradients from a frozen or jointly trained diffusion model through a differentiable renderer, rather than relying solely on photometric reconstruction losses from input images [G].

## Shared principles
All methods in this family optimize a NeRF representation $\theta$ (or latent codes $x$) by minimizing a loss derived from a diffusion model $\phi$ [G]. The core mechanism involves rendering an image (or latent) from the current NeRF state, perturbing it with noise, and computing gradients based on the diffusion model's prediction of that noise [G].

The general optimization loop is:
1. Render an image $x$ from the NeRF at a random viewpoint [G].
2. Sample a diffusion timestep $t$ and Gaussian noise $\epsilon$ [G].
3. Create a noisy version $z_t = \alpha_t x + \sigma_t \epsilon$ [G].
4. Predict noise $\hat{\epsilon} = \hat{\epsilon}_\phi(z_t; \text{condition}, t)$ [G].
5. Compute gradients $\nabla_\theta \mathcal{L}$ using the difference between predicted and actual noise [G].

The specific form of the gradient and the conditioning mechanism varies by child method.

## Subtopics

### [DreamFusion](12087-dreamfusion.md)
Optimizes a NeRF using Score Distillation Sampling (SDS) from a frozen 2D text-to-image diffusion model. It sets itself apart by introducing SDS to backpropagate gradients from a 2D prior to 3D geometry without explicit 3D supervision. Choose it for text-to-3D generation.

### [DreamAvatar](12490-dreamavatar.md)
Generates 3D human avatars using SDS but constrains the NeRF with an SMPL parametric shape prior. It is distinguished by its explicit geometric topology constraints, unlike DreamFusion's implicit representation. Choose it for human avatar generation.

### [Single-Stage Diffusion NeRF Training](12803-single-stage-diffusion-nerf-training.md)
Jointly optimizes scene latent codes, a diffusion prior, and a NeRF decoder in a single loop. It solves the bias of two-stage methods by treating diffusion parameters as trainable. Choose it for multi-view reconstruction with a diffusion prior.

### [ReconFusion](13211-reconfusion.md)
Uses a multiview-conditioned latent diffusion model to regularize few-view reconstruction. It differs from SDS by sampling a new image at each iteration using a denoising U-Net conditioned on input views, reducing volumetric artifacts. Choose it for few-view 3D reconstruction.

### [Nerfbusters](13427-nerfbusters.md)
Applies a local 3D geometric diffusion prior to remove floaters. It distinguishes itself by using a 3D U-Net on voxelized patches (Density Score Distillation Sampling) rather than a 2D image prior. Choose it for cleaning up casually captured NeRFs.

## Background
The concept of distilling 2D diffusion priors into 3D representations was introduced by [DreamFusion](12087-dreamfusion.md). Subsequent works adapted this principle for specific domains: [DreamAvatar](12490-dreamavatar.md) for humans, [Single-Stage Diffusion NeRF Training](12803-single-stage-diffusion-nerf-training.md) for joint optimization, [ReconFusion](13211-reconfusion.md) for few-view reconstruction, and [Nerfbusters](13427-nerfbusters.md) for artifact removal.

## Sources
