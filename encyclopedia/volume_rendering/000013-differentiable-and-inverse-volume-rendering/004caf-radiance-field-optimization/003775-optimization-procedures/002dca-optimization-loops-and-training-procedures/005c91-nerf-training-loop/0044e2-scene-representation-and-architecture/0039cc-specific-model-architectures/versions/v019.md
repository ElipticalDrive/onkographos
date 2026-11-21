# Specific Model Architectures

Also called: NeRF Architectures, Radiance Field Models

## Overview
This family encompasses the specific neural network structures and architectural modifications used to define the radiance field function $f_\theta(\mathbf{x}, \mathbf{d})$ within the NeRF training loop. While the parent article [NeRF Training Loop](970-nerf-training-loop.md) defines the optimization procedure, these architectures determine how spatial coordinates and viewing directions are mapped to volume density $\sigma$ and color $\mathbf{c}$. They differ from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on the network topology, input encoding, and feature aggregation mechanisms rather than the sampling strategy or loss functions.

## Shared principles
All methods in this family compute a radiance field function $f_\theta(\mathbf{x}, \mathbf{d})$ that outputs volume density $\sigma$ and color $\mathbf{c}$ for a query point $\mathbf{x}$ and direction $\mathbf{d}$. The final pixel color $\mathbf{C}(\mathbf{r})$ for a ray $\mathbf{r}$ is computed via the standard volume rendering integral:
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
This family encompasses radiance field models that replace or augment standard Multi-Layer Perceptrons (MLPs) with Transformer architectures to model spatial dependencies, aggregate multi-view information, or process volume tokens. Unlike sibling methods that rely on local convolutions, hash grids, or simple feature fusion, Transformer-based approaches explicitly compute pairwise attention over input tokens, capturing long-range dependencies that local operations miss. They are primarily distinguished by their ability to generalize across scenes (via attention-based aggregation of source views) or to efficiently model large-scale structures in volume super-resolution and denoising.

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

### [ProNeRF](9673-pronerf.md)
ProNeRF is a few-point volumetric rendering method that uses a Projection-Aware Sampling (PAS) network to predict sparse 3D sampling points and opacity adjustments, allowing high-quality rendering with significantly fewer samples than standard NeRF. It achieves faster inference than vanilla NeRF with higher quality than HyperReel by combining a coarse-to-fine sampling strategy with an alternating exploration-exploitation training loop. Unlike generalizable radiance fields that store all training input images for epipolar feature projection, ProNeRF uses a fixed subset of reference views, ensuring a consistent memory footprint.

### [IRON](10251-iron.md)
IRON is a neural inverse rendering pipeline that reconstructs high-quality 3D triangle meshes and spatially varying material textures from photometric images captured with a collocated flashlight and moving camera. It distinguishes itself from sibling methods by employing a hybrid optimization scheme for neural Signed Distance Fields (SDFs): it first uses volumetric radiance field rendering to recover correct object topology, then refines geometry and disentangles materials from lighting using edge-aware physics-based surface rendering. This two-stage approach avoids the local minima and topological errors common in direct surface optimization.

### [pi-GAN](10298-pi-gan.md)
pi-GAN is a generative adversarial approach for unsupervised 3D representation learning from 2D images, synthesizing view-consistent images by conditioning an implicit radiance field on a noise vector. It differs from standard 2D GANs by outputting a 5D spatio-angular radiance field rendered via volume rendering, and from other implicit GANs (like GRAF) by using a StyleGAN-inspired mapping network with Feature-wise Linear Modulation (FiLM) for conditioning and a progressive growing strategy to stabilize training.

#### SinNeRF
- [S1, unofficial] Uses Depth map to constrain unobserved views via Adversarial loss and Perceptual loss.

#### NeRF in the Wild
- [S1, unofficial] Adds an Appearance embedding to express different lighting conditions.
- [S1, unofficial] Adds a Transient embedding to model changing geometry.

#### CLIP-NeRF
- [S2] A method that adds CLIP (a text-image pre-trained multimodal model) to provide greater freedom in shape operations and global deformation, using a disentangled conditional NeRF architecture.
- [S2] 1. Apply a learned deformation network to shape modulation. 2. Embed appearance codes into volume rendering to map color output. 3. Allow control over both shape and appearance.

#### NeRF-RPN
- [S2] Uses a multi-scale Object-NeRF architecture where the scene branch encodes geometry and appearance via a voxel representation of 3D neural volume features, and the object branch encodes each independent object.

#### JacobiNeRF
- [S2] The model synthesizes a NeRF model based on second-order information (Jacobian matrix) to maximize mutual information between highly correlated entities under random scene perturbations. The rendering equation used is: $\hat{C}(r, z_s, z_c) = \sum_{i=1}^{N_c-1} c_i \alpha_i \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$, where $a_i = 1 - \exp(-\sigma_i \delta_i)$ is consistent with discrete volume rendering.

#### DE-NeRF
- [S3] Decomposes lighting, geometry, and appearance to allow relighting of high-frequency environments and appearance editing.
- [S3] Train a signed distance field (SDF) function using a set of 2D images to reconstruct a display grid as the input scene for the MLP.
- [S3] Decompose the scene into geometric features and appearance features (diffuse reflection, roughness, specular tint).
- [S3] Input geometric features, appearance features, and distance h (from sampling point to grid) into four corresponding MLPs to predict diffuse albedo, roughness value, and specular tint.
- [S3] Integrate diffuse reflectance with the environment map to get diffuse color cd.
- [S3] Train a specular lighting MLP Fs to get specular lighting cl.
- [S3] Multiply specular lighting cl with specular hue to get specular color cs.
- [S3] Calculate final color c as cs + cd.
- [S3] On NeRF Synthetic and Shiny Blender data sets, PSNR is 2 points higher than latest papers.

#### RefSR-NeRF
- [S3] Reference-guided super-resolution neural radiation field framework combining NeRF and RefSR-CNN to achieve high-quality new perspective synthesis while balancing memory and speed.
- [S3] Stage 1: Modify NeRF sampling strategy to adopt patch ray sampling on the high-resolution input image to obtain a low-resolution image.
- [S3] RefSR-CNN yields feature maps from the detail-dominant information module and the degradation-dominant information module.
- [S3] Fuse the two feature maps to reconstruct a high-resolution image.
- [S3] On the NeRF-LLFF data set, PSNR is nearly one point higher than the best paper.

#### RelightingNeRF
- [S3] Proposed for handling unstructured photographs and free-viewpoint reconstruction, changing model architecture to output SDF values and latent features.
- [S3] Input sampling point position into the first MLP, which outputs SDF values and latent features.
- [S3] Obtain normals, shadow cues, and specular cues from the SDF values.
- [S3] Provide sampling point positions, lighting positions, and viewing directions, along with latent features from the first MLP, to the second MLP.
- [S3] Second MLP produces the final color.

#### DiffusioNeRF
- [S3] A method that utilizes a denoising diffusion model to learn priors for scene geometry and color, normalizing the geometry and color of the scene to achieve better quality in reconstructed geometry under sparse input views.
- [S3] During NeRF training, render random RGBD patches. Backpropagate the estimated gradients of the log-likelihood to the color and density fields to normalize the geometry and color.
- [S3] On the LLFF dataset, the PSNR indicators in 3, 6, and 9 input views are 1-2 points higher. Among three input views, the PSNR index is nearly 1 point higher than ViP-NeRF.

#### MS-NeRF
- [S3] Addresses rendering distortion and blurring caused by reflection or refraction (e.g., mirrors) where standard NeRF fails multi-view consistency. Represents the scene by a set of characteristic fields in parallel subspaces.
- [S3] Modify NeRF output and volumetric rendering module to output density and features instead of density and color, generating multiple sets of density and features.
- [S3] Volumetric rendering generates multiple feature maps from the multiple sets of density and features.
- [S3] Apply a Decoder MLP in each subspace to decode RGB images.
- [S3] Use a Gate MLP to obtain pixel-level weights for the feature maps.
- [S3] Integrate features from each subspace by summation to obtain the final result.
- [S3] On synthetic, real, and RFFR data sets, the PSNR index is 2-4 points higher.

#### NeRFLight
- [S3] Extends grid-based methods by dividing the scene into eight different regions to split NeRF into separate density and color fields, using a shared feature grid.
- [S3] Avoids increasing the number of features for seamless reconstruction while reducing stored feature values.
- [S3] Divide the scene into eight different regions, each corresponding to an MLP that outputs density and color.
- [S3] Use a shared feature grid with features symmetrically placed at the center of the scene.

#### Edit-NeRF
- [S2] The first method to execute user-controlled deformation models on NeRF, proposing a modular network's conditional radiance field to guide model editing.
- [S2] 1. Train on multiple instances of the same category. 2. Learn underlying part features in an unsupervised manner. 3. Allow user guidance in 2D image shape/texture to propagate to 3D space.
- [S2] Limited capability in shape operations; it only supports adding or deleting local parts of an object. The editing process is slow due to its iterative optimization nature.

#### DT-NeRF
- [S4] Both the Diffusion and Transformer models are trained jointly within a unified optimization framework, sharing the same loss function, and are not pretrained but optimized simultaneously during training through backpropagation.

#### Deformable Radiance Fields
- [S5] Represents a dynamic surgical scene as a static canonical radiance field and a time-dependent displacement field to handle non-rigid deformations.
- [S5] Preprocessing: Map input coordinates and time into Fourier features using positional encoding γ(·) before feeding to the networks.
- [S5] Color and occupancy at time t for point x are retrieved as F_Θ(x + G_Φ(x, t), d).
- [S5] Canonical field F_Θ(x, d): 8-layer MLP with parameters Θ. Inputs: coordinates x ∈ R^3, unit view direction d ∈ R^3. Outputs: RGB color c(x, d) ∈ R^3, space occupancy σ(x) ∈ R.
- [S5] Displacement field G_Φ(x, t): 8-layer MLP with parameters Φ. Inputs: space-time coordinates (x, t). Output: displacement vector between point x at time t and corresponding point in canonical field.

#### NeRF-XL Multi-GPU Representation Strategy
- [S6] A parameter distribution strategy for multi-GPU NeRF training where each GPU handles a distinct 3D tiled region with independent hash encoding and density MLP, but shares the color MLP and appearance/camera embeddings across all GPUs.
- [S6] Hash Encoding: Independent per GPU, confined within a non-overlapping bounding box.
- [S6] Density MLP: Independent per GPU.
- [S6] Color MLP: Shared across all GPUs using DDP to ensure consistent color prediction for novel views.
- [S6] Appearance Embedding: Shared across GPUs; set to zero during novel-view rendering if enabled during training.
- [S6] Camera Pose Embedding: Shared across GPUs if camera optimization is enabled.
- [S6] Unlike independent training where each GPU has its own full NeRF, this method unifies instantiations as a 'single NeRF' with spatially partitioned parameters, increasing model capacity without redundancy.

## Background
The original NeRF architecture, introduced by Mildenhall et al. in 2020, established the baseline of using an 8-layer MLP with positional encoding to map 3D coordinates and viewing directions to density and color [S2]. Subsequent research has expanded this into a diverse family of architectures addressing specific limitations such as training speed, generalization, geometric accuracy, and dynamic scene representation. Key developments include the introduction of hash encoding for efficiency [S6], the use of Transformers for global context [S4], and the integration of generative priors for single-view synthesis [S1, unofficial].

## Sources
[S1] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
[S2] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
[S3] Methods and Strategies for Improving the Novel View Synthesis Quality of Neural Radiation Field, https://doi.org/10.1109/access.2024.3382997, fetched 2026-10-30
[S4] DT-NeRF: A Diffusion and Transformer-Based Optimization Approach for Neural Radiance Fields in 3D Reconstruction, https://doi.org/10.62762/tis.2025.874668, fetched 2026-10-26
[S5] Neural Rendering for Stereo 3D Reconstruction of Deformable Tissues in Robotic Surgery, https://doi.org/10.1007/978-3-031-16449-1_41, fetched 2026-10-26
[S6] NeRF-XL: Scaling NeRFs with Multiple GPUs, https://doi.org/10.1007/978-3-031-73195-2_6, fetched 2026-10-26
