# Specific Model Architectures

Also called: NeRF Architectures, Radiance Field Model Variants

## Overview
Specific Model Architectures (family) encompasses the structural modifications to the canonical Neural Radiance Field (NeRF) MLP and its surrounding inference pipeline to address limitations in training speed, generalization, geometric accuracy, and generative capabilities. While the canonical NeRF uses a single MLP to map 3D coordinates and viewing directions to density and color, this family includes methods that replace the global network with sparse grids, integrate external priors (diffusion, depth, SfM), modify the output to support GANs or light fields, or introduce hierarchical representations for continuous Level of Detail (LOD). It sets itself apart from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on the *network topology and parameterization* rather than the sampling strategy or integration algorithm.

## Shared principles
All methods in this family compute a radiance field function $f_\theta(\mathbf{x}, \mathbf{d})$ that outputs volume density $\sigma$ and color $\mathbf{c}$ (or latent features) for a query point $\mathbf{x}$ and direction $\mathbf{d}$. The final pixel color $\mathbf{C}(\mathbf{r})$ for a ray $\mathbf{r}$ is computed via the standard volume rendering integral:
$$
\mathbf{C}(\mathbf{r}) = \sum_{i=1}^{N} T_i \alpha_i \mathbf{c}_i
$$
where $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$. The architectural variations lie in how $\sigma_i$ and $\mathbf{c}_i$ are generated from the input coordinates, how the network is structured (depth, width, connectivity), and what auxiliary inputs (embeddings, priors, features) are fused into the computation.

## Subtopics

### [AdaNeRF](6195-adanerf.md)
AdaNeRF employs fixed sample positions along each ray and incrementally introduces sparsity during training to achieve real-time rendering. It reduces the number of sample points by training a lightweight Sampling Network to predict importance weights for all points on a line in one go, allowing a heavier Shading Network to process only the most significant samples. This approach differs from siblings like ENeRF, which uses multi-view geometry for surface depth, and NSVF, which uses sparse voxels to skip empty space.

### [AligNeRF](8695-alignerf.md)
AligNeRF is a training strategy and architectural modification for Neural Radiance Fields that addresses rendering artifacts caused by misalignment from moving objects or camera calibration errors, while improving high-frequency detail recovery in high-resolution settings. It sets itself apart from standard NeRF implementations by combining a convolution-augmented architecture with a staged, alignment-aware fine-tuning procedure that utilizes patch-based sampling and optical flow estimation to correct spatial inconsistencies.

### [HyperNeRFGAN](8765-hypernerfgan.md)
HyperNeRFGAN is a generative adversarial network (GAN) architecture that synthesizes 3D-aware radiance field representations from 2D images without requiring camera pose information during training. It solves the limitation of standard NeRF-based generators, which depend on known camera positions, by employing a hypernetwork to generate the weights of a simplified NeRF target network directly from a Gaussian noise vector. Unlike sibling methods such as GRAF or $\pi$-GAN that condition SIREN networks, HyperNeRFGAN uses a hypernetwork paradigm to produce NeRF weights, enabling 3D-aware image generation on datasets where camera estimation is challenging or impossible.

### [MultiPlaneNeRF](8875-multiplanenerf.md)
MultiPlaneNeRF is a radiance field optimization method that replaces the trainable, object-specific neural network weights of vanilla NeRF with a non-trainable 2D image-based representation. It solves the problems of long training times and poor generalization to unseen data by projecting 3D points onto a fixed set of 2D training images to create an aggregate feature vector, which is then processed by a small, shared implicit decoder. Unlike vanilla NeRF, which encodes shape and color in network weights requiring separate training per object, MultiPlaneNeRF allows a single decoder to generalize across many objects by simply swapping the input 2D images.

### [Pool proposer architecture](8953-pool-proposer-architecture.md)
The Pool proposer architecture is a trainable module within the NeRF-ID framework that generates fine-grained sampling positions for volume rendering by processing coarse features. It solves the problem of adaptive ray sampling by learning to propose new points based on the density and color information already gathered at coarse sample locations. Unlike the Transformer proposer, which attends to all coarse features globally, the Pool proposer processes each coarse feature independently before aggregating them, resulting in a simpler, more stable, and computationally cheaper design. It is generally preferred over the Transformer variant due to the latter's instability and poor performance on specific scenes, though it exhibits higher variance than the MLPMix variant.

### [Transformer-based Architectures](8998-transformer-based-architectures.md)
Transformer-based Architectures replace or augment the MLP backbone of NeRF with Transformer modules to capture long-range dependencies and global context. This addresses the limitation of local MLPs in modeling complex scenes with large spatial extents or intricate global structures. By attending to all sampled points or grid cells simultaneously, these architectures improve geometric reconstruction and global consistency, particularly in sparse-view scenarios.

### [Latent-NeRF](9110-latent-nerf.md)
Latent-NeRF is a method that optimizes a Neural Radiance Field (NeRF) representation by leveraging a pre-trained latent diffusion model to guide the rendering process. It solves the problem of generating high-quality novel views from limited input data by applying score distillation within the compact latent space of a pre-trained autoencoder, rather than in pixel space. Unlike standard NeRF training which relies solely on photometric reconstruction loss, Latent-NeRF uses the perceptual priors of a diffusion model to enforce image realism and structural coherence, allowing for effective optimization even with sparse or ambiguous inputs.

### [NeRDi](9112-nerdi.md)
NeRDi is a single-view synthesis method that leverages a pre-trained 2D image diffusion model to provide general image priors, enabling high-quality novel view synthesis from a single input image. Unlike standard NeRF which requires multiple views for optimization, NeRDi minimizes the diffusion loss of rendered views under the constraint of the input view, supplemented by semantic and geometric guidance. It is distinguished by its use of a visual language model to generate conditional inputs for the diffusion process and its integration of estimated depth maps to regularize 3D geometry.

### [Neural Light Field Representations](9273-neural-light-field-representations.md)
Neural Light Field Representations map camera rays directly to pixel colors, bypassing the volumetric ray-marching and alpha-compositing steps required by standard Neural Radiance Fields (NeRF). Unlike NeRF, which predicts density and color at sampled points along a ray, these methods treat the scene as a light field, outputting the final RGB value in a single forward pass. This family is distinguished by its use of distillation from a pre-trained NeRF teacher to a deep student network, enabling inference speeds orders of magnitude faster than iterative sampling methods.

### [RING-NeRF](9323-ring-nerf.md)
RING-NeRF is a neural field architecture that introduces two inductive biases: a continuous multi-scale representation of the scene and an invariance of the decoder’s latent space over spatial and scale domains. It solves the problem of rendering quality degradation at varying observation distances and resolutions by providing intrinsic continuous Level of Detail (LOD) without explicit LOD-specific supervision. Unlike sibling methods such as AdaNeRF or Zip-NeRF which require explicit supervision for LOD or modify features based on scale, RING-NeRF encodes LOD exclusively in the grid hierarchy and uses a distance-aware forward mapping to adapt to observation distance, avoiding aliasing artifacts.

### [Sparse and Efficient Representations](9326-sparse-and-efficient-representations.md)
This family addresses the prohibitive memory and computational costs of dense Neural Radiance Fields (NeRFs) in large-scale or high-resolution scenes by decomposing the global radiance field into localized, sparse components. Unlike dense representations that query a single global network for every point, these methods partition space or features to enable parallel training, reduce per-point latency, and allow for efficient mesh extraction. They distinguish themselves from sibling architectures like AdaNeRF or Transformer-based Architectures by explicitly leveraging spatial sparsity or hierarchical decomposition rather than adaptive feature hashing or attention mechanisms alone.

### [Geometric Priors and Depth Integration](9395-geometric-priors-and-depth-integration.md)
This family of methods enhances Neural Radiance Fields (NeRF) by integrating explicit geometric constraints—such as Structure-from-Motion (SfM) features, epipolar geometry, or monocular depth priors—into the optimization loop. This addresses the shape-radiance ambiguity and local minima issues inherent in standard NeRF, particularly under sparse-view conditions. It differs from sibling methods like AdaNeRF or Latent-NeRF by explicitly enforcing 3D geometric consistency (depth ordering, feature matching) rather than relying solely on appearance-based regularization or latent space compression.

### [Anisotropic Neural Representation Learning](9475-anisotropic-neural-representation-learning.md)
Anisotropic Neural Representation Learning is a method that models volumetric density and latent features as view-dependent functions using spherical harmonics (SH) to eliminate directional ambiguity in neural rendering, replacing the isotropic point-sampled features of standard NeRF. Unlike Mip-NeRF which samples cones to approximate pre-filtered representations, this method modifies the MLP to predict view-dependent SH coefficients directly, avoiding the need for shape-based sampling parameterization. It is designed as a plug-and-play sub-model that can be integrated into various NeRF-based frameworks to improve reconstruction quality and reduce shape-radiance ambiguity.

### SinNeRF
SinNeRF uses a depth map to constrain unobserved views via Adversarial loss and Perceptual loss [S1, unofficial].

### NeRF in the Wild
NeRF in the Wild adds an Appearance embedding to express different lighting conditions and a Transient embedding to model changing geometry [S1, unofficial].

### StyleNeRF
StyleNeRF is a 3D-aware generative model that merges 3D scene representation with style-based generative models [S2]. It uses NeRF to generate low-resolution feature maps and samples them to high resolution to avoid super-computation volume rendering, and applies regularization terms to improve 3D consistency [S2].

### CLIP-NeRF
CLIP-NeRF is a method that adds CLIP (a text-image pre-trained multimodal model) to provide greater freedom in shape operations and global deformation, using a disentangled conditional NeRF architecture [S2]. It applies a learned deformation network to shape modulation, embeds appearance codes into volume rendering to map color output, and allows control over both shape and appearance [S2].

### NeRF-RPN
NeRF-RPN uses a multi-scale Object-NeRF architecture where the scene branch encodes geometry and appearance via a voxel representation of 3D neural volume features, and the object branch encodes each independent object [S2].

### JacobiNeRF
JacobiNeRF synthesizes a NeRF model based on second-order information (Jacobian matrix) to maximize mutual information between highly correlated entities under random scene perturbations. The rendering equation used is: $\hat{C}(r, z_s, z_c) = \sum_{i=1}^{N_c-1} c_i \alpha_i \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$, where $a_i = 1 - \exp(-\sigma_i \delta_i)$ is consistent with discrete volume rendering [S2].

### DE-NeRF
DE-NeRF decomposes lighting, geometry, and appearance to allow relighting of high-frequency environments and appearance editing [S3]. It trains a signed distance field (SDF) function using a set of 2D images to reconstruct a display grid as the input scene for the MLP. It decomposes the scene into geometric features and appearance features (diffuse reflection, roughness, specular tint). It inputs geometric features, appearance features, and distance $h$ (from sampling point to grid) into four corresponding MLPs to predict diffuse albedo, roughness value, and specular tint. It integrates diffuse reflectance with the environment map to get diffuse color $c_d$. It trains a specular lighting MLP $F_s$ to get specular lighting $c_l$. It multiplies specular lighting $c_l$ with specular hue to get specular color $c_s$. It calculates final color $c$ as $c_s + c_d$. On NeRF Synthetic and Shiny Blender data sets, PSNR is 2 points higher than latest papers [S3].

### RefSR-NeRF
RefSR-NeRF is a reference-guided super-resolution neural radiation field framework combining NeRF and RefSR-CNN to achieve high-quality new perspective synthesis while balancing memory and speed [S3]. Stage 1: Modify NeRF sampling strategy to adopt patch ray sampling on the high-resolution input image to obtain a low-resolution image. RefSR-CNN yields feature maps from the detail-dominant information module and the degradation-dominant information module. It fuses the two feature maps to reconstruct a high-resolution image. On the NeRF-LLFF data set, PSNR is nearly one point higher than the best paper [S3].

### RelightingNeRF
RelightingNeRF is proposed for handling unstructured photographs and free-viewpoint reconstruction, changing model architecture to output SDF values and latent features [S3]. Input sampling point position into the first MLP, which outputs SDF values and latent features. Obtain normals, shadow cues, and specular cues from the SDF values. Provide sampling point positions, lighting positions, and viewing directions, along with latent features from the first MLP, to the second MLP. Second MLP produces the final color [S3].

### DiffusioNeRF
DiffusioNeRF is a method that utilizes a denoising diffusion model to learn priors for scene geometry and color, normalizing the geometry and color of the scene to achieve better quality in reconstructed geometry under sparse input views [S3]. During NeRF training, render random RGBD patches. Backpropagate the estimated gradients of the log-likelihood to the color and density fields to normalize the geometry and color. On the LLFF dataset, the PSNR indicators in 3, 6, and 9 input views are 1-2 points higher. Among three input views, the PSNR index is nearly 1 point higher than ViP-NeRF [S3].

### MS-NeRF
MS-NeRF addresses rendering distortion and blurring caused by reflection or refraction (e.g., mirrors) where standard NeRF fails multi-view consistency. It represents the scene by a set of characteristic fields in parallel subspaces [S3]. Modify NeRF output and volumetric rendering module to output density and features instead of density and color, generating multiple sets of density and features. Volumetric rendering generates multiple feature maps from the multiple sets of density and features. Apply a Decoder MLP in each subspace to decode RGB images. Use a Gate MLP to obtain pixel-level weights for the feature maps. Integrate features from each subspace by summation to obtain the final result. On synthetic, real, and RFFR data sets, the PSNR index is 2-4 points higher [S3].

### NeRFLight
NeRFLight extends grid-based methods by dividing the scene into eight different regions to split NeRF into separate density and color fields, using a shared feature grid. It avoids increasing the number of features for seamless reconstruction while reducing stored feature values [S3]. Divide the scene into eight different regions, each corresponding to an MLP that outputs density and color. Use a shared feature grid with features symmetrically placed at the center of the scene [S3].

### Edit-NeRF
Edit-NeRF is the first method to execute user-controlled deformation models on NeRF, proposing a modular network's conditional radiance field to guide model editing [S2]. Train on multiple instances of the same category. Learn underlying part features in an unsupervised manner. Allow user guidance in 2D image shape/texture to propagate to 3D space. Limited capability in shape operations; it only supports adding or deleting local parts of an object. The editing process is slow due to its iterative optimization nature [S2].

### DT-NeRF
Both the Diffusion and Transformer models are trained jointly within a unified optimization framework, sharing the same loss function, and are not pretrained but optimized simultaneously during training through backpropagation [S4].

## Background
The canonical NeRF architecture, introduced by Mildenhall et al. in 2020 [S1, unofficial][S2], established the baseline of using a single MLP to map 3D coordinates and viewing directions to density and color. Subsequent research rapidly diversified the architectural landscape to address specific limitations:
*   **Efficiency and Sparsity:** Methods like AdaNeRF [S1, unofficial] and Sparse and Efficient Representations [S3] emerged to reduce computational costs.
*   **Generative Capabilities:** HyperNeRFGAN [S1, unofficial][S2] and StyleNeRF [S2] integrated GANs and hypernetworks to enable 3D-aware image generation without camera poses.
*   **Geometric Accuracy:** Geometric Priors [S3] and Anisotropic Neural Representation Learning [S1, unofficial] addressed shape-radiance ambiguity.
*   **Single-View Synthesis:** Latent-NeRF [S3] and NeRDi [S1, unofficial][S2] leveraged pre-trained diffusion models to guide NeRF optimization from limited inputs.
*   **High-Resolution and Detail:** AligNeRF [S3] and RefSR-NeRF [S3] introduced convolutional post-processing and patch-based sampling for high-fidelity rendering.
*   **Hierarchical Representation:** RING-NeRF [S1, unofficial] introduced continuous LOD for multi-scale rendering.

These developments reflect a shift from a single monolithic network to modular, hybrid, and prior-guided architectures tailored to specific application constraints.

## Sources
[S1] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
[S2] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
[S3] Methods and Strategies for Improving the Novel View Synthesis Quality of Neural Radiation Field, https://doi.org/10.1109/access.2024.3382997, fetched 2026-10-30
[S4] DT-NeRF: A Diffusion and Transformer-Based Optimization Approach for Neural Radiance Fields in 3D Reconstruction, https://doi.org/10.62762/tis.2025.874668, fetched 2026-10-26
