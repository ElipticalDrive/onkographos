# Diffusion-guided Optimization
Also called: Score Distillation Sampling, Latent Diffusion NeRF, Local 3D Diffusion Prior

## Overview
Diffusion-guided optimization methods enhance Neural Radiance Field (NeRF) training by integrating pretrained 2D or 3D diffusion models as priors to regularize geometry, fill missing views, or generate novel 3D content from text. Unlike standard NeRF training which relies solely on photometric reconstruction losses from observed images, these methods leverage the generative knowledge of diffusion models to constrain the optimization landscape [S1, S2, S3]. They are distinguished from sibling methods like [KNN-NeRF](8672-knn-nerf.md) or [Cross-view Self-enhancement](13434-cross-view-self-enhancement.md) by their explicit use of score distillation or latent diffusion losses rather than nearest-neighbor statistics or self-supervised consistency alone.

## Shared principles
All methods in this family share the core mechanism of using a diffusion model to provide gradient signals for optimizing NeRF parameters or latent scene representations. The fundamental operation involves rendering an image (or latent) from the current NeRF state, perturbing it with noise, and computing a gradient based on the diffusion model's prediction of the noise or score. This gradient is backpropagated through the differentiable renderer to update the NeRF.

Key shared components include:
1.  **Differentiable Rendering:** The NeRF must support differentiable volume rendering to compute gradients with respect to its parameters $\theta$ or latent codes $x$.
2.  **Diffusion Prior:** A pretrained model (2D image-to-image or 3D voxel-to-voxel) that predicts noise $\epsilon_\theta$ or scores given a noisy input and conditioning (text, images, or features).
3.  **Distillation Loss:** A loss function derived from the diffusion model's objective, typically approximating the gradient of the log-probability of the generated data under the diffusion process. Common forms include Score Distillation Sampling (SDS) [S2] or Density Score Distillation Sampling (DSDS) [S1].
4.  **Optimization Loop:** An iterative process where the NeRF is updated using gradients from both the diffusion prior and, where applicable, reconstruction losses from input images.

## Subtopics

### [DreamFusion](12087-dreamfusion.md)
DreamFusion optimizes a NeRF from scratch using Score Distillation Sampling (SDS) to generate 3D objects from text prompts. It operates directly in pixel space, rendering images and adding noise to compute SDS gradients. It is chosen when text-to-3D generation is required without existing 3D data. Unlike other methods, it does not use latent codes or multi-stage training.

### [Single-Stage Diffusion NeRF Training](12803-single-stage-diffusion-nerf-training.md)
SSDNeRF jointly optimizes a NeRF decoder, diffusion prior, and latent scene codes (e.g., triplanes) in a single loop. It uses a cached gradient strategy to balance the high cost of diffusion loss with rendering loss. It is chosen for few-view reconstruction to avoid the bias and noise of two-stage methods. It differs from DreamFusion by operating in latent space and using a unified training loop.

### [ReconFusion](13211-reconfusion.md)
ReconFusion regularizes NeRF optimization with a multiview-conditioned diffusion model trained for novel view synthesis. It samples new images at each iteration using DDIM and classifier-free guidance. It is chosen for few-view reconstruction to prevent catastrophic failures and reduce floaters. It differs from SDS by using a multiview-conditioned prior and sampling-based loss rather than score distillation.

### [Nerfbusters](13427-nerfbusters.md)
Nerfbusters employs a local 3D geometric prior learned via a diffusion model on synthetic surface patches. It uses Density Score Distillation Sampling (DSDS) on voxelized density cubes. It is chosen to remove ghostly artifacts (floaters) from casually captured images. It differs from global priors by being category-agnostic, locally applied, and explicitly 3D-consistent.

### DreamAvatar
DreamAvatar is a diffusion-based 3D avatar generation method that uses an SMPL model as a shape prior and text-guided 2D image generation via diffusion to create training data in a DreamFusion-like pipeline [S1]. It combines NeRF and diffusion to generate avatars with text guidance.

## Background
The concept of using diffusion models for 3D generation emerged in 2022-2023, building on the success of 2D text-to-image models like Imagen and Stable Diffusion. DreamFusion introduced Score Distillation Sampling (SDS) to distill 2D generative knowledge into 3D NeRFs [S2]. Subsequent works like SSDNeRF and ReconFusion addressed limitations such as computational cost and the need for explicit 3D data by introducing latent spaces and multiview conditioning [S1]. Nerfbusters introduced local 3D priors to address specific artifacts like floaters in casual captures [S1]. These methods are widely used in 3D content creation, virtual reality, and digital twin generation.

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
