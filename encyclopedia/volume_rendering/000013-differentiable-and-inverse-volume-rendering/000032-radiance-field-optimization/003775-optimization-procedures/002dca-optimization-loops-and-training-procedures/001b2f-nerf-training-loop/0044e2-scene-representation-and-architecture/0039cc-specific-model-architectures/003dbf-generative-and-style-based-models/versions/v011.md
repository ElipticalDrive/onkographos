# Generative and Style-based Models

Also called: 3D-aware generative models, NeRF-GAN hybrids

## Overview
Generative and Style-based Models integrate Neural Radiance Fields (NeRF) with generative frameworks such as Generative Adversarial Networks (GANs) or diffusion models to synthesize novel 3D-consistent views from limited or unstructured data. Unlike standard NeRF which reconstructs a specific scene from multi-view images, these models learn a distribution of scenes or generate high-fidelity textures that standard NeRF cannot resolve due to resolution limits or lack of input views. They are distinguished from siblings like [AdaNeRF](6195-adanerf.md) by their primary objective of *generation* rather than *reconstruction*, often employing hypernetworks or latent space decoders to bypass the computational cost of high-resolution volumetric rendering.

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

### [HyperNeRFGAN](8765-hypernerfgan.md)
HyperNeRFGAN is a generative adversarial network (GAN) architecture that employs a hypernetwork to transform a Gaussian noise vector into the weights of a simplified NeRF target network, enabling 3D-aware image generation without requiring camera position information during training. Unlike standard NeRF which uses a 5D input (position and viewing direction), HyperNeRFGAN uses a 3D input (position only) because the training datasets lack view-dependent characteristics like reflections. It differs from siblings like [Latent-NeRF](9110-latent-nerf.md) by using a hypernetwork to generate NeRF weights directly, avoiding the complexity of NeRF conditioning mechanisms while maintaining 3D awareness.

### [Latent-NeRF](9110-latent-nerf.md)
Latent-NeRF is a method for optimizing Neural Radiance Fields (NeRF) by mapping the radiance field output to a compact latent space that is subsequently decoded by a pre-trained 2D diffusion model. It solves the problem of low-resolution details and poor generalization in standard NeRF training by leveraging the high-fidelity generative priors of Stable Diffusion. Unlike standard NeRF which predicts RGB colors directly, Latent-NeRF predicts latent features, allowing the use of score distillation from a diffusion model to enforce photorealistic textures and structural consistency. It is distinguished from its sibling [HyperNeRFGAN](8765-hypernerfgan.md) by using a diffusion-based generative prior rather than a GAN-based adversarial loss.

### StyleNeRF
**Definition:** A 3D-aware generative model that merges 3D scene representation with style-based generative models [S1].
**Steps:** Uses NeRF to generate low-resolution feature maps and samples them to high resolution to avoid super-computation volume rendering. Applies regularization terms to improve 3D consistency [S1].

### NeRFFaceShop
**Definition:** A 3D-aware generative model that produces animatable and relightable human heads by training on large-scale in-the-wild videos [S2].
**Steps:** Deforms feature maps rather than 3D points so lighting and expression changes stay consistent during motion. Builds on a tri-plane generator and introduces a unified animation and lighting representation. Extracts expression coefficients from video to learn a detailed animation space, enabling smooth, controllable head motion and relighting across humans and nonhuman domains [S2].

### Generative Latent Optimization
**Definition:** A method where a set of randomly sampled latent codes is paired to a set of images, and the latent codes are input to a generator whose parameters are jointly optimized with the latent code using a reconstruction loss. It can be thought of as a Discriminator-less GAN [S2].
**Parameters:** $u_i$ represents the other inputs not optimized over (needed in NeRF but not necessarily for other models) [S2].

### LOLNeRF
**Definition:** A method capable of single-shot view synthesis of human faces, built similarly to $\pi$-GAN but using Generative Latent Optimization instead of adversarial training [S2].

## Background
The intersection of NeRF and generative modeling began with GRAF (Schwarz et al., 2020), which established the foundation for combining conditional radiance fields with GANs to generate high-resolution synthetic views, though initially limited to single-object scenes [S1]. To address multi-object scenes and the lack of high-quality images in earlier GAN-NeRF hybrids, GIRAFFE (Niemeyer et al., 2021) introduced compositional generative neural feature fields, separating shape, appearance, and background without explicit supervision [S1]. Subsequently, $\pi$-GAN (Chan et al., 2021) utilized periodic activation functions and volume rendering to achieve view-consistent radiance fields for high-quality 3D-aware image synthesis, though it required substantial data and was best suited for simple geometries like faces and cars [S1].

Later developments focused on efficiency and specific domains. StyleNeRF (Gu et al., 2022) merged 3D scene representation with style-based generation, using NeRF for low-resolution feature maps and upsampling to avoid the computational cost of full high-resolution volume rendering, while applying regularization for 3D consistency [S1]. In the domain of human faces, NeRFFaceShop (2025) learned animatable and relightable models from in-the-wild videos by deforming feature maps rather than 3D points, ensuring lighting and expression consistency [S2]. LOLNeRF (2021) achieved single-shot view synthesis of faces using Generative Latent Optimization, offering a discriminator-less alternative to adversarial training [S2]. These methods collectively address the limitations of standard NeRF in generalization, resolution, and data efficiency.

## Sources
[S1] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
[S2] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
