# Specific Model Architectures

Also called: NeRF Architectures, Radiance Field Models

## Overview
Specific Model Architectures refers to the family of neural network structures and representations used to parameterize the radiance field $f_\theta(\mathbf{x}, \mathbf{d})$ within the NeRF training loop. While the standard NeRF uses a simple MLP mapping 3D coordinates and viewing directions to density and color, this family encompasses modifications to the network topology, input representations, and output mechanisms to address limitations such as training speed, generalization, memory efficiency, and handling of complex phenomena like lighting changes or reflections. These architectures set themselves apart from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on the *function* $f_\theta$ itself rather than the *sampling* strategy or *loss* functions.

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
Not given in the sources.

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

### SinNeRF
Uses Depth map to constrain unobserved views via Adversarial loss and Perceptual loss [S1, unofficial].

### NeRF in the Wild
Adds an Appearance embedding to express different lighting conditions. Adds a Transient embedding to model changing geometry [S1, unofficial].

### StyleNeRF
A 3D-aware generative model that merges 3D scene representation with style-based generative models. Uses NeRF to generate low-resolution feature maps and samples them to high resolution to avoid super-computation volume rendering. Applies regularization terms to improve 3D consistency [S2].

### CLIP-NeRF
A method that adds CLIP (a text-image pre-trained multimodal model) to provide greater freedom in shape operations and global deformation, using a disentangled conditional NeRF architecture. 1. Apply a learned deformation network to shape modulation. 2. Embed appearance codes into volume rendering to map color output. 3. Allow control over both shape and appearance [S2].

### NeRF-RPN
Uses a multi-scale Object-NeRF architecture where the scene branch encodes geometry and appearance via a voxel representation of 3D neural volume features, and the object branch encodes each independent object [S2].

### JacobiNeRF
The model synthesizes a NeRF model based on second-order information (Jacobian matrix) to maximize mutual information between highly correlated entities under random scene perturbations. The rendering equation used is: $\hat{C}(r, z_s, z_c) = \sum_{i=1}^{N_c-1} c_i \alpha_i \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$, where $a_i = 1 - \exp(-\sigma_i \delta_i)$ is consistent with discrete volume rendering [S2].

### DE-NeRF
Decomposes lighting, geometry, and appearance to allow relighting of high-frequency environments and appearance editing. Train a signed distance field (SDF) function using a set of 2D images to reconstruct a display grid as the input scene for the MLP. Decompose the scene into geometric features and appearance features (diffuse reflection, roughness, specular tint). Input geometric features, appearance features, and distance h (from sampling point to grid) into four corresponding MLPs to predict diffuse albedo, roughness value, and specular tint. Integrate diffuse reflectance with the environment map to get diffuse color cd. Train a specular lighting MLP Fs to get specular lighting cl. Multiply specular lighting cl with specular hue to get specular color cs. Calculate final color c as cs + cd. On NeRF Synthetic and Shiny Blender data sets, PSNR is 2 points higher than latest papers [S3].

### RefSR-NeRF
Reference-guided super-resolution neural radiation field framework combining NeRF and RefSR-CNN to achieve high-quality new perspective synthesis while balancing memory and speed. Stage 1: Modify NeRF sampling strategy to adopt patch ray sampling on the high-resolution input image to obtain a low-resolution image. RefSR-CNN yields feature maps from the detail-dominant information module and the degradation-dominant information module. Fuse the two feature maps to reconstruct a high-resolution image. On the NeRF-LLFF data set, PSNR is nearly one point higher than the best paper [S3].

### RelightingNeRF
Proposed for handling unstructured photographs and free-viewpoint reconstruction, changing model architecture to output SDF values and latent features. Input sampling point position into the first MLP, which outputs SDF values and latent features. Obtain normals, shadow cues, and specular cues from the SDF values. Provide sampling point positions, lighting positions, and viewing directions, along with latent features from the first MLP, to the second MLP. Second MLP produces the final color [S3].

### DiffusioNeRF
A method that utilizes a denoising diffusion model to learn priors for scene geometry and color, normalizing the geometry and color of the scene to achieve better quality in reconstructed geometry under sparse input views. During NeRF training, render random RGBD patches. Backpropagate the estimated gradients of the log-likelihood to the color and density fields to normalize the geometry and color. On the LLFF dataset, the PSNR indicators in 3, 6, and 9 input views are 1-2 points higher. Among three input views, the PSNR index is nearly 1 point higher than ViP-NeRF [S3].

### MS-NeRF
Addresses rendering distortion and blurring caused by reflection or refraction (e.g., mirrors) where standard NeRF fails multi-view consistency. Represents the scene by a set of characteristic fields in parallel subspaces. Modify NeRF output and volumetric rendering module to output density and features instead of density and color, generating multiple sets of density and features. Volumetric rendering generates multiple feature maps from the multiple sets of density and features. Apply a Decoder MLP in each subspace to decode RGB images. Use a Gate MLP to obtain pixel-level weights for the feature maps. Integrate features from each subspace by summation to obtain the final result. On synthetic, real, and RFFR data sets, the PSNR index is 2-4 points higher [S3].

### NeRFLight
Extends grid-based methods by dividing the scene into eight different regions to split NeRF into separate density and color fields, using a shared feature grid. Avoids increasing the number of features for seamless reconstruction while reducing stored feature values. Divide the scene into eight different regions, each corresponding to an MLP that outputs density and color. Use a shared feature grid with features symmetrically placed at the center of the scene [S3].

### Edit-NeRF
The first method to execute user-controlled deformation models on NeRF, proposing a modular network's conditional radiance field to guide model editing. Train on multiple instances of the same category. Learn underlying part features in an unsupervised manner. Allow user guidance in 2D image shape/texture to propagate to 3D space. Limited capability in shape operations; it only supports adding or deleting local parts of an object. The editing process is slow due to its iterative optimization nature [S2].

## Background
The original NeRF architecture, introduced by Mildenhall et al. in 2020, established the baseline MLP-based radiance field representation [S1, unofficial][S2]. Subsequent research rapidly expanded the architectural landscape to address specific limitations. Early extensions like NeRF in the Wild (2021) introduced embeddings for lighting and transient objects to handle unconstrained photo collections [S1, unofficial]. Generative approaches like StyleNeRF (2022) and HyperNeRFGAN merged NeRF with GANs to enable 3D-aware generation without camera poses [S2, S3]. Efficiency-focused architectures like AdaNeRF and Sparse representations emerged to tackle the computational bottlenecks of dense volumetric rendering [S1, unofficial][S3]. More recent works integrate pre-trained diffusion models (Latent-NeRF, NeRDi) and geometric priors (SinNeRF, RegNeRF) to improve quality under sparse-view conditions [S1, unofficial][S3].

## Sources
[S1] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
[S2] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
[S3] Methods and Strategies for Improving the Novel View Synthesis Quality of Neural Radiation Field, https://doi.org/10.1109/access.2024.3382997, fetched 2026-10-30
