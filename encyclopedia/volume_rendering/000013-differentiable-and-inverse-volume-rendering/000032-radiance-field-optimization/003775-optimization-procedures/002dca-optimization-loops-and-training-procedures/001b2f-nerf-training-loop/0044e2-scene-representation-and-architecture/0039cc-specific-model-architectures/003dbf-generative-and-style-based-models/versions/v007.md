# Generative and Style-based Models

Also called: Generative NeRF, Style-based NeRF, 3D-aware NeRF

## Overview
Generative and Style-based Models extend parametric radiance fields from inverse rendering (reconstructing a specific scene) to generative modeling (synthesizing novel 3D-consistent scenes from noise or text). They solve the problem of creating high-quality, view-consistent 3D data when explicit 3D supervision or dense multi-view inputs are unavailable. Unlike standard NeRFs that optimize for a single static scene, these models learn a distribution over scenes, often integrating GANs or diffusion priors to guide the optimization of geometry and appearance [S1].

## Shared principles
The core mechanism remains the parametric function $F_\Theta$ that maps a 3D position $\mathbf{x} \in \mathbb{R}^3$ and a viewing direction $\mathbf{d} \in \mathbb{R}^3$ to a volume density $\sigma \in \mathbb{R}$ and an RGB color $\mathbf{c} \in \mathbb{R}^3$. This function is typically implemented as one or more neural networks.

**Input Representation:**
Raw coordinates are rarely used directly due to the high-frequency nature of natural images. They are mapped to a higher-dimensional space using positional encoding $\gamma(\mathbf{x})$. For a coordinate $\mathbf{x}$, the encoding includes the original value and sinusoidal functions at increasing frequencies:
$$
\gamma(\mathbf{x}) = (\sin(2^0 \pi \mathbf{x}), \cos(2^0 \pi \mathbf{x}), \dots, \sin(2^{L-1} \pi \mathbf{x}), \cos(2^{L-1} \pi \mathbf{x}))
$$
where $L$ is the number of frequency bands [Parent].

**Network Topology:**
The canonical implementation uses two coupled Multi-Layer Perceptrons (MLPs): a density network $F_\sigma$ and a color network $F_c$ [Parent].
1.  **Density Network:** Takes the encoded position $\gamma(\mathbf{x})$ as input. It consists of $L$ hidden layers with ReLU activations and skip connections from the input to every 4th layer to preserve low-frequency information [Parent]. The output is the density $\sigma$.
2.  **Color Network:** Takes the encoded position $\gamma(\mathbf{x})$, the raw viewing direction $\mathbf{d}$, and the intermediate feature vector from the density network (usually the output of the last hidden layer before the density head) as input. It predicts the color $\mathbf{c}$ using a final ReLU or sigmoid activation [Parent].

**Rendering Integration:**
During training, rays are sampled at points $\mathbf{x}_i$ along the ray. The MLPs predict $\sigma_i$ and $\mathbf{c}_i$. The pixel color $C(\mathbf{r})$ is computed via volumetric rendering:
$$
C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
$$
and $\delta_i$ is the distance between samples [Parent].

## Subtopics

### [HyperNeRFGAN](8765-hypernerfgan.md)
HyperNeRFGAN is a generative model that employs a hypernetwork to produce the weights of a simplified NeRF target network, trained using the StyleGAN2 objective to generate 3D-aware representations from a Gaussian noise distribution. It solves the problem of 3D-aware image generation when camera positions are unavailable or difficult to estimate, which is common in medical imaging and datasets with limited viewpoints. Unlike standard NeRFs that require viewing directions, HyperNeRFGAN omits direction inputs, relying on a single MLP that maps only spatial location to color and density. It distinguishes itself from siblings like [Latent-NeRF](9110-latent-nerf.md) by using a hypernetwork to directly generate NeRF weights via Factorized Multiplicative Modulation (FMM) layers, rather than conditioning a SIREN-based network or using latent codes to modulate a fixed architecture.

### [Latent-NeRF](9110-latent-nerf.md)
Latent-NeRF is a method for optimizing Neural Radiance Field (NeRF) representations by integrating pre-trained latent diffusion models, enabling high-quality novel view synthesis and text-guided scene generation from sparse or single-view inputs. It distinguishes itself from standard NeRF training by replacing the direct RGB color output with a compact latent feature space, allowing the use of score distillation from a stable diffusion model to guide the optimization of geometry and appearance. Unlike HyperNeRFGAN, which uses generative adversarial networks for hypernetwork weight prediction, Latent-NeRF directly optimizes the NeRF parameters via gradients derived from a diffusion model's denoising process.

### StyleNeRF
- **Definition:** A 3D-aware generative model that merges 3D scene representation with style-based generative models [S1].
- **Mechanism:** Uses NeRF to generate low-resolution feature maps and samples them to high resolution to avoid super-computation volume rendering [S1]. Applies regularization terms to improve 3D consistency [S1].

## Background
Generative NeRF models emerged from the convergence of Neural Radiance Fields (NeRF) and Generative Adversarial Networks (GANs). The foundation was laid in 2020 by Schwarz et al. with GRAF, which introduced conditional radiance fields to control shape and appearance, though limited to single-object scenes [S1]. In 2021, Niemeyer et al. proposed GIRAFFE, which represented scenes as compositional generative neural feature fields, separating object shape, appearance, and background without explicit supervision [S1]. Also in 2021, Chan et al. introduced pi-GAN, using periodic activation functions and volume rendering to achieve view-consistent radiance fields for high-quality 3D-aware image synthesis [S1]. These methods, however, required large datasets and were often limited to simple geometries like faces or cars [S1]. Subsequent works like StyleNeRF (Gu et al., 2022) and HyperNeRFGAN further refined these approaches by integrating style-based generation and hypernetworks to handle complex scenes and limited viewpoints [S1].

## Sources
[S1] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
