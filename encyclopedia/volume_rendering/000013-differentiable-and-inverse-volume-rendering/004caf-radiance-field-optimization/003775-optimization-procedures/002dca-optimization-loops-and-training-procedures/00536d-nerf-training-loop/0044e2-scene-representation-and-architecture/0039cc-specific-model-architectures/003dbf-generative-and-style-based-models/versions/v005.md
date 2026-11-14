# Generative and Style-based Models

Also called: Generative NeRF, Style-based NeRF, 3D-aware GANs for NeRF

## Overview
This family integrates Neural Radiance Fields (NeRF) with generative adversarial networks (GANs) or diffusion models to synthesize novel 3D-consistent images from noise or latent codes, rather than optimizing a single static scene from posed images. Unlike standard NeRF which reconstructs a specific observed scene, these models learn a distribution of 3D scenes. They set themselves apart from siblings like [AdaNeRF](6195-adanerf.md) (which adapts a fixed NeRF to new views via attention) by generating the 3D representation itself—either through weight modulation, latent space optimization, or architectural fusion—enabling 3D-aware image generation.

## Shared principles
The family builds on [Specific Model Architectures](8585-specific-model-architectures.md). The core distinction is the replacement of the deterministic optimization loop with a generative objective. While the canonical representation maps $\mathbf{x}$ and $\mathbf{d}$ to $\sigma$ and $\mathbf{c}$, generative variants modify this mapping to be conditional on a latent variable $\mathbf{z}$ or to output weights/features compatible with a generative prior. The rendering equation remains:
$$
C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
$$
However, the inputs to the networks or the loss functions driving the training differ significantly, often removing explicit viewing directions or replacing photometric loss with perceptual/diffusion losses.

## Subtopics

### [HyperNeRFGAN](8765-hypernerfgan.md)
HyperNeRFGAN generates 3D-aware images by using a hypernetwork to produce the weights of a target NeRF. It solves the problem of 3D generation without camera poses by omitting viewing directions from the NeRF input. It sets itself apart by generating *weights* via Factorized Multiplicative Modulation (FMM) rather than conditioning a fixed network.

### [Latent-NeRF](9110-latent-nerf.md)
Latent-NeRF optimizes a NeRF representation using score distillation from a pre-trained diffusion model. It solves the problem of generating high-quality views from limited data by replacing photometric loss with diffusion guidance. It sets itself apart by outputting compact latent features (4 channels) instead of RGB, which are then denoised by Stable Diffusion.

### StyleNeRF
StyleNeRF merges 3D scene representation with style-based generative models to achieve 3D-aware generation. It generates low-resolution feature maps using NeRF and upsamples them to high resolution to avoid the computational cost of full volumetric rendering at high resolution. It applies regularization terms to improve 3D consistency and addresses issues related to view conditioning and noise injection [S1].

## Background
The integration of NeRF and GANs began with GRAF (Schwarz et al., 2020), which established the foundation for conditional radiance fields but was limited to single-object scenes [S1]. Subsequent works like GIRAFFE (Niemeyer et al., 2021) introduced compositional generation for multi-object scenes, and pi-GAN (Chan et al., 2021) utilized periodic activation functions for high-quality 3D-aware synthesis [S1]. However, these methods often required large datasets and struggled with complex, realistic scenes [S1]. StyleNeRF was proposed in 2022 to address computational efficiency and 3D consistency by combining NeRF with style-based generation [S1].

## Sources
[S1] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
