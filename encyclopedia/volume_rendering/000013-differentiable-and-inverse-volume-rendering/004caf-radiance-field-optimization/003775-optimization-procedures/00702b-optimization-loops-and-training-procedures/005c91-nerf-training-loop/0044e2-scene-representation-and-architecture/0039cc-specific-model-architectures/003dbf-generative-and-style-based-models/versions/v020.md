# Generative and Style-based Models

Also called: Generative NeRF, 3D-aware GANs, Style-based Radiance Fields

## Overview
Generative and Style-based Models extend the NeRF paradigm from reconstructing a single static scene to synthesizing novel, high-fidelity 3D-consistent images from a distribution of scenes [G]. They solve the problem of limited training data and the inability of standard NeRF to generalize across a class of objects by conditioning the radiance field on a latent code [G]. Unlike standard NeRF, which optimizes weights for a fixed scene, these models learn a mapping from a latent space to the scene geometry and appearance, enabling the generation of new instances [G]. They are distinguished from siblings like [AdaNeRF](6195-adanerf.md) and [SinNeRF](6199-sinnerf.md) by their explicit use of generative priors (GANs, Diffusion) and style-based conditioning to control appearance attributes and ensure multi-view consistency without requiring explicit 3D supervision [G].

## Shared principles
All methods in this family share the mechanism of mapping 6D input coordinates (3D position $\mathbf{x}$ and 2D viewing direction $\mathbf{d}$) to 4D output values (density $\sigma$ and color $\mathbf{c}$) using a differentiable neural network [G]. This allows the radiance field to be optimized through the volumetric rendering integral [G].

**Input Representation:**
Raw coordinates $\mathbf{x}$ and $\mathbf{d}$ are rarely used directly due to the spectral bias of MLPs, which struggle to fit high-frequency details [G]. They are mapped to a higher-dimensional space using positional encoding $\gamma(\cdot)$ [G]. For a coordinate $\mathbf{x}$, the encoding includes the original value and sinusoidal functions at increasing frequencies:
$$
\gamma(\mathbf{x}) = (\sin(2^0 \pi \mathbf{x}), \cos(2^0 \pi \mathbf{x}), \dots, \sin(2^{L-1} \pi \mathbf{x}), \cos(2^{L-1} \pi \mathbf{x}))
$$
where $L$ is the number of frequency bands [G].

**Network Topology:**
The canonical implementation uses two coupled Multi-Layer Perceptrons (MLPs): a density network $F_\sigma$ and a color network $F_c$ [G].
1.  **Density Network:** Takes the encoded position $\gamma(\mathbf{x})$ as input. It consists of $L$ hidden layers with ReLU activations and skip connections from the input to every 4th layer to preserve low-frequency information [G]. The output is the density $\sigma$.
2.  **Color Network:** Takes the encoded position $\gamma(\mathbf{x})$, the raw viewing direction $\mathbf{d}$, and the intermediate feature vector from the density network (usually the output of the last hidden layer before the density head) as input. It predicts the color $\mathbf{c}$ [G].

**Rendering Integration:**
During training, rays are sampled at points $\mathbf{x}_i$ along the ray. The MLPs predict $\sigma_i$ and $\mathbf{c}_i$. The pixel color $C(\mathbf{r})$ is computed via volumetric rendering:
$$
C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
$$
where $\delta_i$ is the distance between samples [G].

## Subtopics

### [StyleNeRF](8659-stylenerf.md)
StyleNeRF is a 3D-aware generative model that integrates a neural radiance field (NeRF) into a style-based generator framework to synthesize photo-realistic, high-resolution images with high multi-view consistency from unstructured 2D images. It solves the computational bottleneck of standard NeRF-based generative models by performing volume rendering only at a low resolution to produce feature maps, which are then upsampled in 2D space to high resolution, avoiding the super-computational cost of rendering every pixel via full volume integration. Unlike its siblings [HyperNeRFGAN](8765-hypernerfgan.md), [Latent-NeRF](9110-latent-nerf.md), [pi-GAN](10298-pi-gan.md), and [GIRAFFE](11360-giraffe.md), StyleNeRF explicitly conditions the NeRF MLP weights on style vectors derived from a mapping network, enabling fine-grained control over appearance attributes while maintaining 3D geometry consistency.

### [HyperNeRFGAN](8765-hypernerfgan.md)
HyperNeRFGAN is a generative adversarial network (GAN) architecture that employs a hypernetwork to transform a Gaussian noise vector into the weights of a simplified NeRF target network, enabling 3D-aware image generation without requiring camera position information during training. Unlike standard NeRF which uses a 5D input (position and viewing direction), HyperNeRFGAN uses a 3D input (position only) because the training datasets lack view-dependent characteristics like reflections. It differs from siblings like [Latent-NeRF](9110-latent-nerf.md) by using a hypernetwork to generate NeRF weights directly, avoiding the complexity of NeRF conditioning mechanisms while maintaining 3D awareness.

### [Latent-NeRF](9110-latent-nerf.md)
Latent-NeRF is a method for optimizing Neural Radiance Fields (NeRF) by mapping scene geometry and appearance into a compact latent space, which is then refined using a pre-trained 2D latent diffusion model. It solves the problem of low-frequency artifacts and lack of high-frequency detail in standard NeRF training by leveraging the generative priors of diffusion models. Unlike standard NeRF, which predicts RGB colors directly, Latent-NeRF predicts pseudo-color channels corresponding to latent features, allowing the use of powerful 2D image priors for 3D consistency.

### [pi-GAN](10298-pi-gan.md)
pi-GAN is a generative adversarial network (GAN) approach to unsupervised 3D representation learning from 2D images. It conditions an implicit radiance field, represented by a SIREN network, on a noise vector to produce a 5D spatio-angular radiance field, which is rendered via volume rendering to synthesize view-consistent images. It is distinguished from its sibling [StyleNeRF](8659-stylenerf.md) and [GRAF](9572-generative-and-style-based-models.md) by using a StyleGAN-inspired mapping network with Feature-wise Linear Modulation (FiLM) to condition the entire SIREN backbone, rather than concatenating latent codes or using separate shape/appearance codes. It outperformed GRAF in image quality and achieved state-of-the-art results on CelebA, Cats, and CARLA datasets.

### [GIRAFFE](11360-giraffe.md)
GIRAFFE represents scenes as compositions of $N$ entities (objects plus a background), where each entity possesses a separate neural feature field and an affine transformation. This architecture enables explicit control over the pose, shape, and appearance of individual entities within a generated scene. It distinguishes itself from sibling methods like [StyleNeRF](8659-stylenerf.md) and [HyperNeRFGAN](8765-hypernerfgan.md) by decoupling scene composition from the generative process, allowing for variable numbers of objects and independent manipulation of entity attributes via latent codes and transformation parameters.

### [Generative Latent Optimization](11963-generative-latent-optimization.md)
Generative Latent Optimization (GLO) is a training procedure for implicit scene representations where a set of latent codes is jointly optimized with the neural network parameters to minimize a reconstruction loss between rendered outputs and target images. It functions as a discriminator-less GAN, allowing the model to learn a mapping from a fixed latent distribution to the scene space without adversarial training. Unlike standard NeRF training which optimizes only network weights for a fixed scene, GLO treats the latent codes as learnable parameters, enabling the model to capture view-dependent variations or specific scene instances.

### NeRFFaceShop
A 3D-aware generative model that produces animatable and relightable human heads by training on large-scale in-the-wild videos [S1]. It deforms feature maps rather than 3D points so lighting and expression changes stay consistent during motion [S1]. It builds on a tri-plane generator and introduces a unified animation and lighting representation [S1]. It extracts expression coefficients from video to learn a detailed animation space, enabling smooth, controllable head motion and relighting across humans and nonhuman domains [S1].

### [Adversarial Training (GAN)](14030-adversarial-training-gan.md)
Adversarial Training integrates a discriminator network trained in parallel with a generator (e.g., a radiance field or super-resolution network) to classify rendered images as real or fake. It distinguishes itself from standard differentiable rendering pipelines by adding a game-theoretic loss term that encourages photorealism and high-frequency detail, addressing the blurriness often associated with pure pixel-wise reconstruction losses.

### UPST-NeRF
A method for 3D photorealistic style transfer that leverages a hypernetwork to control features of style images as latent codes for scene stylization [S2]. HyperNet takes features extracted by StyleEncoder with VGG as input to update the weights of HyperLiner, followed by modifying RGBNet information to update the scene style [S2]. It trains a 2D PST network for more realistic scene presentation [S2].

### ARF
Introduces a nearest neighbor feature matching loss to impart detailed style structures onto the original NeRF [S2]. It employs a deferred back-propagation technique to optimize with full-resolution images, overcoming GPU memory constraints [S2].

### StyleNeRF / GIRAFFE HD
Retain 3D controllability of GIRAFFE while generating higher quality/resolution images (>512x512) using the StyleGAN2 architecture [S3].

## Background
Generative NeRF models emerged as an extension of the original NeRF paper [G], aiming to generalize radiance fields from single-scene reconstruction to multi-scene generation. Early works like [GRAF](9572-generative-and-style-based-models.md) and [pi-GAN](10298-pi-gan.md) introduced GAN-based frameworks to learn 3D-aware representations from 2D images [G]. Subsequent developments such as [StyleNeRF](8659-stylenerf.md) and [HyperNeRFGAN](8765-hypernerfgan.md) incorporated style-based generators and hypernetworks to improve resolution and controllability [G]. More recent approaches like [Latent-NeRF](9110-latent-nerf.md) integrate diffusion models for high-fidelity detail synthesis [G]. These methods are widely used in computer graphics for style transfer [S2, S3].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
[S3] Semantically-aware Neural Radiance Fields for Visual Scene Understanding: A Comprehensive Review, https://link.springer.com/article/10.1007/s11263-025-02663-5, fetched 2026-10-27
