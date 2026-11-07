# Specific Model Architectures

Also called: NeRF Architectures, Radiance Field Models

## Overview
Specific Model Architectures refers to the family of neural network designs used to parameterize the radiance field function $f_\theta(\mathbf{x}, \mathbf{d})$ within the NeRF optimization loop. While the standard NeRF implementation uses a simple Multi-Layer Perceptron (MLP) to map 3D coordinates and viewing directions to density and color, this family encompasses modifications to that core mapping to address specific limitations such as training speed, generalization, geometric ambiguity, and resolution constraints. These architectures differ from their siblings in the training loop (such as [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) or [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md)) by defining *what* is being learned and *how* the spatial data is structured, rather than *how* the samples are selected or integrated.

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

### [MultiPlaneNeRF](8875-multiplanenerf.md)
MultiPlaneNeRF is a radiance field optimization method that replaces the trainable, object-specific neural network weights of vanilla NeRF with a non-trainable 2D image-based representation. It solves the problems of long training times and poor generalization to unseen data by projecting 3D points onto a fixed set of 2D training images to create an aggregate feature vector, which is then processed by a small, shared implicit decoder. Unlike vanilla NeRF, which encodes shape and color in network weights requiring separate training per object, MultiPlaneNeRF allows a single decoder to generalize across many objects by simply swapping the input 2D images.

### [Pool proposer architecture](8953-pool-proposer-architecture.md)
The Pool proposer architecture is a trainable module within the NeRF-ID framework that generates fine-grained sampling positions for volume rendering by processing coarse features. It solves the problem of adaptive ray sampling by learning to propose new points based on the density and color information already gathered at coarse sample locations. Unlike the Transformer proposer, which attends to all coarse features globally, the Pool proposer processes each coarse feature independently before aggregating them, resulting in a simpler, more stable, and computationally cheaper design. It is generally preferred over the Transformer variant due to the latter's instability and poor performance on specific scenes, though it exhibits higher variance than the MLPMix variant.

### [Transformer-based Architectures](8998-transformer-based-architectures.md)
Transformer-based Architectures integrate attention mechanisms into the radiance field representation to capture long-range dependencies and global context. These models improve geometric reconstruction and handle complex scenes by processing features across the entire scene or large patches simultaneously, rather than relying solely on local MLP queries. They are chosen when global consistency is critical and computational resources allow for the higher overhead of attention operations.

### [NeRDi](9112-nerdi.md)
NeRDi is a single-view synthesis method that leverages a pre-trained 2D image diffusion model to provide general image priors, enabling high-quality novel view synthesis from a single input image. Unlike standard NeRF which requires multiple views for optimization, NeRDi minimizes the diffusion loss of rendered views under the constraint of the input view, supplemented by semantic and geometric guidance. It is distinguished by its use of a visual language model to generate conditional inputs for the diffusion process and its integration of estimated depth maps to regularize 3D geometry.

### [Neural Light Field Representations](9273-neural-light-field-representations.md)
Neural Light Field Representations map camera rays directly to pixel colors, bypassing the volumetric ray-marching and alpha-compositing steps required by standard Neural Radiance Fields (NeRF). Unlike NeRF, which predicts density and color at sampled points along a ray, these methods treat the scene as a light field, outputting the final RGB value in a single forward pass. This family is distinguished by its use of distillation from a pre-trained NeRF teacher to a deep student network, enabling inference speeds orders of magnitude faster than iterative sampling methods.

### [RING-NeRF](9323-ring-nerf.md)
RING-NeRF is a neural field architecture that introduces two inductive biases: a continuous multi-scale representation of the scene and an invariance of the decoder’s latent space over spatial and scale domains. It solves the problem of rendering quality degradation at varying observation distances and resolutions by providing intrinsic continuous Level of Detail (LOD) without explicit LOD-specific supervision. Unlike sibling methods such as [AdaNeRF](6195-adanerf.md) or [Zip-NeRF](8875-multiplanenerf.md) which require explicit supervision for LOD or modify features based on scale, RING-NeRF encodes LOD exclusively in the grid hierarchy and uses a distance-aware forward mapping to adapt to observation distance, avoiding aliasing artifacts.

### [Sparse and Efficient Representations](9326-sparse-and-efficient-representations.md)
This family addresses the prohibitive memory and computational costs of dense Neural Radiance Fields (NeRFs) in large-scale or high-resolution scenes by decomposing the global radiance field into localized, sparse components. Unlike dense representations that query a single global network for every point, these methods partition space or features to enable parallel training, reduce per-point latency, and allow for efficient mesh extraction. They distinguish themselves from sibling architectures like [AdaNeRF](6195-adanerf.md) or [Transformer-based Architectures](8998-transformer-based-architectures.md) by explicitly leveraging spatial sparsity or hierarchical decomposition rather than adaptive feature hashing or attention mechanisms alone.

### [Geometric Priors and Depth Integration](9395-geometric-priors-and-depth-integration.md)
This family of methods enhances Neural Radiance Fields (NeRF) by integrating explicit geometric constraints—such as Structure-from-Motion (SfM) features, epipolar geometry, or monocular depth priors—into the optimization loop. This addresses the shape-radiance ambiguity and local minima issues inherent in standard NeRF, particularly under sparse-view conditions. It differs from sibling methods like [AdaNeRF](6195-adanerf.md) or [Latent-NeRF](9110-latent-nerf.md) by explicitly enforcing 3D geometric consistency (depth ordering, feature matching) rather than relying solely on appearance-based regularization or latent space compression.

### [Anisotropic Neural Representation Learning](9475-anisotropic-neural-representation-learning.md)
Anisotropic Neural Representation Learning is a method that models volumetric density and latent features as view-dependent functions using spherical harmonics (SH) to eliminate directional ambiguity in neural rendering, replacing the isotropic point-sampled features of standard NeRF. Unlike Mip-NeRF which samples cones to approximate pre-filtered representations, this method modifies the MLP to predict view-dependent SH coefficients directly, avoiding the need for shape-based sampling parameterization. It is designed as a plug-and-play sub-model that can be integrated into various NeRF-based frameworks to improve reconstruction quality and reduce shape-radiance ambiguity.

### [Generative and Style-based Models](9572-generative-and-style-based-models.md)
This family integrates generative adversarial networks (GANs) or diffusion models with Neural Radiance Fields to synthesize novel 3D-consistent images from noise or text, rather than optimizing a single static scene from photographs. Unlike standard NeRF training which fits a specific dataset, these models learn a distribution of 3D shapes and textures. They are distinguished from siblings like [AdaNeRF](6195-adanerf.md) (which adapts a pre-trained NeRF to new views) by generating the radiance field parameters or latent codes directly from a generative prior, enabling the creation of novel objects.

### SinNeRF
SinNeRF uses a depth map to constrain unobserved views via Adversarial loss and Perceptual loss [S1, unofficial].

### NeRF in the Wild
NeRF in the Wild adds an Appearance embedding to express different lighting conditions and a Transient embedding to model changing geometry [S1, unofficial].

### CLIP-NeRF
CLIP-NeRF is a method that adds CLIP (a text-image pre-trained multimodal model) to provide greater freedom in shape operations and global deformation, using a disentangled conditional NeRF architecture [S2]. It applies a learned deformation network to shape modulation, embeds appearance codes into volume rendering to map color output, and allows control over both shape and appearance [S2].

### NeRF-RPN
NeRF-RPN uses a multi-scale Object-NeRF architecture where the scene branch encodes geometry and appearance via a voxel representation of 3D neural volume features, and the object branch encodes each independent object [S2].

### JacobiNeRF
JacobiNeRF synthesizes a NeRF model based on second-order information (Jacobian matrix) to maximize mutual information between highly correlated entities under random scene perturbations [S2]. The rendering equation used is: $\hat{C}(r, z_s, z_c) = \sum_{i=1}^{N_c-1} c_i \alpha_i \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$, where $a_i = 1 - \exp(-\sigma_i \delta_i)$ is consistent with discrete volume rendering [S2].

### DE-NeRF
DE-NeRF decomposes lighting, geometry, and appearance to allow relighting of high-frequency environments and appearance editing [S3]. It trains a signed distance field (SDF) function using a set of 2D images to reconstruct a display grid as the input scene for the MLP. It decomposes the scene into geometric features and appearance features (diffuse reflection, roughness, specular tint). Input geometric features, appearance features, and distance h (from sampling point to grid) into four corresponding MLPs to predict diffuse albedo, roughness value, and specular tint. It integrates diffuse reflectance with the environment map to get diffuse color cd. It trains a specular lighting MLP Fs to get specular lighting cl. It multiplies specular lighting cl with specular hue to get specular color cs. It calculates final color c as cs + cd. On NeRF Synthetic and Shiny Blender data sets, PSNR is 2 points higher than latest papers [S3].

### RefSR-NeRF
RefSR-NeRF is a reference-guided super-resolution neural radiation field framework combining NeRF and RefSR-CNN to achieve high-quality new perspective synthesis while balancing memory and speed [S3]. In Stage 1, it modifies NeRF sampling strategy to adopt patch ray sampling on the high-resolution input image to obtain a low-resolution image. RefSR-CNN yields feature maps from the detail-dominant information module and the degradation-dominant information module. It fuses the two feature maps to reconstruct a high-resolution image. On the NeRF-LLFF data set, PSNR is nearly one point higher than the best paper [S3].

### RelightingNeRF
RelightingNeRF is proposed for handling unstructured photographs and free-viewpoint reconstruction, changing model architecture to output SDF values and latent features [S3]. It inputs sampling point position into the first MLP, which outputs SDF values and latent features. It obtains normals, shadow cues, and specular cues from the SDF values. It provides sampling point positions, lighting positions, and viewing directions, along with latent features from the first MLP, to the second MLP. The second MLP produces the final color [S3].

### DiffusioNeRF
DiffusioNeRF is a method that utilizes a denoising diffusion model to learn priors for scene geometry and color, normalizing the geometry and color of the scene to achieve better quality in reconstructed geometry under sparse input views [S3]. During NeRF training, it renders random RGBD patches. It backpropagates the estimated gradients of the log-likelihood to the color and density fields to normalize the geometry and color. On the LLFF dataset, the PSNR indicators in 3, 6, and 9 input views are 1-2 points higher. Among three input views, the PSNR index is nearly 1 point higher than ViP-NeRF [S3].

### MS-NeRF
MS-NeRF addresses rendering distortion and blurring caused by reflection or refraction (e.g., mirrors) where standard NeRF fails multi-view consistency. It represents the scene by a set of characteristic fields in parallel subspaces [S3]. It modifies NeRF output and volumetric rendering module to output density and features instead of density and color, generating multiple sets of density and features. Volumetric rendering generates multiple feature maps from the multiple sets of density and features. It applies a Decoder MLP in each subspace to decode RGB images. It uses a Gate MLP to obtain pixel-level weights for the feature maps. It integrates features from each subspace by summation to obtain the final result. On synthetic, real, and RFFR data sets, the PSNR index is 2-4 points higher [S3].

### NeRFLight
NeRFLight extends grid-based methods by dividing the scene into eight different regions to split NeRF into separate density and color fields, using a shared feature grid [S3]. It avoids increasing the number of features for seamless reconstruction while reducing stored feature values. It divides the scene into eight different regions, each corresponding to an MLP that outputs density and color. It uses a shared feature grid with features symmetrically placed at the center of the scene [S3].

### Edit-NeRF
Edit-NeRF is the first method to execute user-controlled deformation models on NeRF, proposing a modular network's conditional radiance field to guide model editing [S2]. It trains on multiple instances of the same category. It learns underlying part features in an unsupervised manner. It allows user guidance in 2D image shape/texture to propagate to 3D space. It has limited capability in shape operations; it only supports adding or deleting local parts of an object. The editing process is slow due to its iterative optimization nature [S2].

### DT-NeRF
In DT-NeRF, both the Diffusion and Transformer models are trained jointly within a unified optimization framework, sharing the same loss function, and are not pretrained but optimized simultaneously during training through backpropagation [S4].

## Background
The original NeRF architecture, introduced by Mildenhall et al. in 2020 [S1, unofficial], established the baseline MLP-based radiance field representation. Subsequent research rapidly expanded the architectural landscape to address specific challenges:
*   **Generalization and Single-View Synthesis:** Methods like NeRDi [S1, unofficial][S2] and SinNeRF [S1, unofficial] integrated pre-trained models (diffusion, VLMs) to enable synthesis from limited inputs.
*   **Efficiency and Speed:** AdaNeRF [S1, unofficial] and Neural Light Field Representations [S1, unofficial] focused on reducing computational overhead through sparsity and direct regression.
*   **Geometric Accuracy:** Anisotropic Neural Representation Learning [S1, unofficial] and Geometric Priors [S1, unofficial] addressed the inherent shape-radiance ambiguity of standard NeRF.
*   **High-Resolution and Detail:** AligNeRF [S1, unofficial][S2] and RefSR-NeRF [S3] introduced convolutional and patch-based architectures to handle high-frequency details and misalignment.
*   **Generative Capabilities:** Generative and Style-based Models [S1, unofficial] and CLIP-NeRF [S2] extended NeRF from reconstruction to generation and editing.

## Sources
[S1] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
[S2] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
[S3] Methods and Strategies for Improving the Novel View Synthesis Quality of Neural Radiation Field, https://doi.org/10.1109/access.2024.3382997, fetched 2026-10-30
[S4] DT-NeRF: A Diffusion and Transformer-Based Optimization Approach for Neural Radiance Fields in 3D Reconstruction, https://doi.org/10.62762/tis.2025.874668, fetched 2026-10-26
