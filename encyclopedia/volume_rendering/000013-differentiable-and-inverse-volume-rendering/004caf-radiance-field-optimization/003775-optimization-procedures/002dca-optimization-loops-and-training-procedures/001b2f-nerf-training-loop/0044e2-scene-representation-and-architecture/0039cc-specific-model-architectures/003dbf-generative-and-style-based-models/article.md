# Generative and Style-based Models

Also called: Generative NeRF, 3D-aware GANs, Style-based NeRF

## Overview
Generative and Style-based Models extend the NeRF paradigm from deterministic scene reconstruction to probabilistic 3D-aware image synthesis. Instead of optimizing a single radiance field for a specific dataset, these methods learn a distribution over 3D scenes, allowing the generation of novel, view-consistent images from latent codes [S1, S2]. They distinguish themselves from sibling methods like [AdaNeRF](6195-adanerf.md) or [CLIP-NeRF](8661-clip-nerf.md) by treating the radiance field as a generator in a GAN or diffusion framework, prioritizing high-fidelity texture synthesis and 3D consistency over exact reconstruction of input views [S2].

## Shared principles
All methods in this family share the mechanism of mapping 6D input coordinates (3D position $\mathbf{x}$ and 2D viewing direction $\mathbf{d}$) to 4D output values (density $\sigma$ and color $\mathbf{c}$) using a differentiable neural network. This allows the radiance field to be optimized through the volumetric rendering integral.

**Input Representation:**
Raw coordinates $\mathbf{x}$ and $\mathbf{d}$ are rarely used directly due to the spectral bias of MLPs, which struggle to fit high-frequency details. They are mapped to a higher-dimensional space using positional encoding $\gamma(\cdot)$ [S1][S3]. For a coordinate $\mathbf{x}$, the encoding includes the original value and sinusoidal functions at increasing frequencies:
$$
\gamma(\mathbf{x}) = (\sin(2^0 \pi \mathbf{x}), \cos(2^0 \pi \mathbf{x}), \dots, \sin(2^{L-1} \pi \mathbf{x}), \cos(2^{L-1} \pi \mathbf{x}))
$$
where $L$ is the number of frequency bands [S1][S3].

**Network Topology:**
The canonical implementation uses two coupled Multi-Layer Perceptrons (MLPs): a density network $F_\sigma$ and a color network $F_c$ [S1][S3].
1.  **Density Network:** Takes the encoded position $\gamma(\mathbf{x})$ as input. It consists of $L$ hidden layers with ReLU activations and skip connections from the input to every 4th layer to preserve low-frequency information [S1][S3]. The output is the density $\sigma$.
2.  **Color Network:** Takes the encoded position $\gamma(\mathbf{x})$, the raw viewing direction $\mathbf{d}$, and the intermediate feature vector from the density network (usually the output of the last hidden layer before the density head) as input. It predicts the color $\mathbf{c}$ using a final ReLU or sigmoid activation [S1][S3].

**Rendering Integration:**
During training, rays are sampled at points $\mathbf{x}_i$ along the ray. The MLPs predict $\sigma_i$ and $\mathbf{c}_i$. The pixel color $C(\mathbf{r})$ is computed via volumetric rendering:
$$
C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
$$
and $\delta_i$ is the distance between samples [S1][S3].

## Subtopics

### [StyleNeRF](8659-stylenerf.md)
StyleNeRF is a 3D-aware generative model that integrates the 3D scene representation of Neural Radiance Fields (NeRF) with the style-based image synthesis framework of StyleGAN. It focuses on high-quality 2D image synthesis by bringing 3D geometric awareness to the generator, producing realistic images and consistent 3D geometry from latent codes. It distinguishes itself from sibling methods like HyperNeRFGAN and Latent-NeRF by explicitly coupling a style-code-conditioned NeRF with an upsampling module to avoid the computational cost of high-resolution volume rendering, while employing specific regularization to ensure 3D consistency.

### [HyperNeRFGAN](8765-hypernerfgan.md)
HyperNeRFGAN is a generative adversarial network (GAN) architecture that employs a hypernetwork to transform a Gaussian noise vector into the weights of a simplified NeRF target network, enabling 3D-aware image generation without requiring camera position information during training. Unlike standard NeRF which uses a 5D input (position and viewing direction), HyperNeRFGAN uses a 3D input (position only) because the training datasets lack view-dependent characteristics like reflections. It differs from siblings like [Latent-NeRF](9110-latent-nerf.md) by using a hypernetwork to generate NeRF weights directly, avoiding the complexity of NeRF conditioning mechanisms while maintaining 3D awareness.

### [Latent-NeRF](9110-latent-nerf.md)
Latent-NeRF is a method for optimizing Neural Radiance Fields (NeRF) by mapping the radiance field output to a compact latent space that is subsequently decoded by a pre-trained 2D diffusion model. It solves the problem of low-resolution details and poor generalization in standard NeRF training by leveraging the high-fidelity generative priors of Stable Diffusion. Unlike standard NeRF which predicts RGB colors directly, Latent-NeRF predicts latent features, allowing the use of score distillation from a diffusion model to enforce photorealistic textures and structural consistency. It is distinguished from its sibling [HyperNeRFGAN](8765-hypernerfgan.md) by using a diffusion-based generative prior rather than a GAN-based adversarial loss.

### [pi-GAN](10298-pi-gan.md)
pi-GAN is a generative adversarial network (GAN) approach to unsupervised 3D representation learning from 2D images. It conditions an implicit radiance field, represented by a SIREN network, on a noise vector to produce a 5D spatio-angular radiance field, which is rendered via volume rendering to synthesize view-consistent images. It is distinguished from its sibling [StyleNeRF](8659-stylenerf.md) and [GRAF](9572-generative-and-style-based-models.md) by using a StyleGAN-inspired mapping network with Feature-wise Linear Modulation (FiLM) to condition the entire SIREN backbone, rather than concatenating latent codes or using separate shape/appearance codes. It outperformed GRAF in image quality and achieved state-of-the-art results on CelebA, Cats, and CARLA datasets.

### [GIRAFFE](11360-giraffe.md)
GIRAFFE represents scenes as compositions of $N$ entities (objects plus a background), where each entity possesses a separate neural feature field and an affine transformation. This architecture enables explicit control over the pose, shape, and appearance of individual entities within a generated scene. It distinguishes itself from sibling methods like [StyleNeRF](8659-stylenerf.md) and [HyperNeRFGAN](8765-hypernerfgan.md) by decoupling scene composition from the generative process, allowing for variable numbers of objects and independent manipulation of entity attributes via latent codes and transformation parameters.

### NeRFFaceShop
- [S1] A 3D-aware generative model that produces animatable and relightable human heads by training on large-scale in-the-wild videos.
- [S1] Deforms feature maps rather than 3D points so lighting and expression changes stay consistent during motion.
- [S1] Builds on a tri-plane generator and introduces a unified animation and lighting representation.
- [S1] Extracts expression coefficients from video to learn a detailed animation space, enabling smooth, controllable head motion and relighting across humans and nonhuman domains.

### Generative Latent Optimization
- [S1] A method where a set of randomly sampled latent codes is paired to a set of images, and the latent codes are input to a generator whose parameters are jointly optimized with the latent code using a reconstruction loss. It can be thought of as a Discriminator-less GAN.
- [S1] u_i represents the other inputs not optimized over (needed in NeRF but not necessarily for other models).

### LOLNeRF
- [S1] A method capable of single-shot view synthesis of human faces, built similarly to π-GAN but using Generative Latent Optimization instead of adversarial training.

## Background
The family originates from the convergence of Neural Radiance Fields (NeRF) and Generative Adversarial Networks (GANs). Early works like pi-GAN [S2] and StyleNeRF [S1] demonstrated that NeRF architectures could be conditioned on latent variables to generate novel 3D-consistent images, moving beyond single-scene reconstruction. HyperNeRFGAN [S1] further simplified this by using hypernetworks to generate NeRF weights directly. More recently, Latent-NeRF [S2] integrated diffusion models to overcome the resolution and texture limitations of GAN-based approaches. These methods are primarily used in 3D-aware image synthesis, avatar generation, and unsupervised 3D representation learning from 2D datasets [S1, S2].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
