# Generative and Style-based Models

Also called: NeRF-GAN, 3D-aware NeRF, Latent-NeRF, HyperNeRFGAN

## Overview
This family extends standard NeRF architectures to enable 3D-aware image generation, allowing the synthesis of novel views and scenes from noise vectors or text prompts rather than requiring a fixed set of input images for optimization [S1]. Unlike standard NeRFs that optimize geometry and appearance for a specific scene, these models learn a distribution over 3D scenes, solving the problem of generating high-quality, view-consistent 3D content from limited or no input data [S1]. They distinguish themselves from siblings like [Transformer-based Architectures](8998-transformer-based-architectures.md) by explicitly integrating generative objectives (GANs or Diffusion) with the volumetric rendering pipeline, often modifying the network topology to support style modulation or latent space optimization [S1, S2, S3].

## Shared principles
The core mechanism remains the differentiable approximation of the continuous radiance field using a neural network or collection of networks, allowing gradient-based optimization [Parent, S1, S3].

**Input Representation:**
Raw coordinates $\mathbf{x}$ and $\mathbf{d}$ are rarely used directly due to the spectral bias of MLPs. They are mapped to a higher-dimensional space using positional encoding $\gamma(\cdot)$ [Parent, S1, S3]. For a coordinate $\mathbf{x}$, the encoding includes the original value and sinusoidal functions at increasing frequencies:
$$
\gamma(\mathbf{x}) = (\sin(2^0 \pi \mathbf{x}), \cos(2^0 \pi \mathbf{x}), \dots, \sin(2^{L-1} \pi \mathbf{x}), \cos(2^{L-1} \pi \mathbf{x}))
$$
where $L$ is the number of frequency bands [Parent, S1, S3].

**Network Topology:**
The canonical implementation uses two coupled Multi-Layer Perceptrons (MLPs): a density network $F_\sigma$ and a color network $F_c$ [Parent, S1, S3].
1.  **Density Network:** Takes the encoded position $\gamma(\mathbf{x})$ as input. It consists of $L$ hidden layers with ReLU activations and skip connections from the input to every 4th layer to preserve low-frequency information [Parent, S1, S3]. The output is the density $\sigma$.
2.  **Color Network:** Takes the encoded position $\gamma(\mathbf{x})$, the raw viewing direction $\mathbf{d}$, and the intermediate feature vector from the density network (usually the output of the last hidden layer before the density head) as input. It predicts the color $\mathbf{c}$ using a final ReLU or sigmoid activation [Parent, S1, S3].

**Rendering Integration:**
During training, rays are sampled at points $\mathbf{x}_i$ along the ray. The MLPs predict $\sigma_i$ and $\mathbf{c}_i$. The pixel color $C(\mathbf{r})$ is computed via volumetric rendering:
$$
C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
$$
and $\delta_i$ is the distance between samples [Parent, S1, S3].

## Subtopics

### [HyperNeRFGAN](8765-hypernerfgan.md)
HyperNeRFGAN is a generative model that employs a hypernetwork to produce the weights of a simplified NeRF target network, trained using the StyleGAN2 objective to generate 3D-aware representations from a Gaussian noise distribution. It solves the problem of 3D-aware image generation when camera positions are unavailable or difficult to estimate, which is common in medical imaging and datasets with limited viewpoints. Unlike standard NeRFs that require viewing directions, HyperNeRFGAN omits direction inputs, relying on a single MLP that maps only spatial location to color and density. It distinguishes itself from siblings like [Latent-NeRF](9110-latent-nerf.md) by using a hypernetwork to directly generate NeRF weights via Factorized Multiplicative Modulation (FMM) layers, rather than conditioning a SIREN-based network or using latent codes to modulate a fixed architecture.

### [Latent-NeRF](9110-latent-nerf.md)
Latent-NeRF is a method for optimizing Neural Radiance Field (NeRF) representations by integrating pre-trained latent diffusion models, enabling high-quality novel view synthesis and text-guided scene generation from sparse or single-view inputs. It distinguishes itself from standard NeRF training by replacing the direct RGB color output with a compact latent feature space, allowing the use of score distillation from a stable diffusion model to guide the optimization of geometry and appearance. Unlike HyperNeRFGAN, which uses generative adversarial networks for hypernetwork weight prediction, Latent-NeRF directly optimizes the NeRF parameters via gradients derived from a diffusion model's denoising process.

### StyleNeRF
- **Definition:** A 3D-aware generative model that merges 3D scene representation with style-based generative models [S1].
- **Mechanism:** Uses NeRF to generate low-resolution feature maps and samples them to high resolution to avoid super-computation volume rendering [S1]. Applies regularization terms to improve 3D consistency [S1].

## Background
The intersection of NeRF and generative models began with GRAF (Schwarz et al., 2020), which introduced a discriminator to generate high-resolution views but was limited to single-object scenes [S1]. This was extended by GIRAFFE (Niemeyer et al., 2021), which used compositional neural feature fields to separate shape, appearance, and background for multi-object scenes [S1]. pi-GAN (Chan et al., 2021) introduced periodic activation functions for high-quality 3D-aware synthesis [S1]. StyleNeRF (Gu et al., 2022) specifically addressed computational costs by combining NeRF with style-based generation, using low-resolution feature maps upsampled to high resolution [S1]. These methods address the limitation of standard NeRFs requiring dense input views, enabling generation from noise or text.

## Sources
[S1] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
