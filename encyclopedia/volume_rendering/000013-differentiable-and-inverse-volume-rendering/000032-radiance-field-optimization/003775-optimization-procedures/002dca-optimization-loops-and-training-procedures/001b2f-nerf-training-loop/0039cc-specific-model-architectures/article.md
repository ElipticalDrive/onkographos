# Specific Model Architectures

Also called: NeRF Architectures, Radiance Field Network Variants

## Overview
Specific Model Architectures refers to the family of modifications to the canonical Neural Radiance Field (NeRF) network topology, input/output representations, and parameterization strategies. While the standard NeRF uses a single Multi-Layer Perceptron (MLP) to map 3D coordinates and viewing directions to density and color, this family encompasses methods that replace, decompose, or augment this core component to address limitations in training speed, rendering latency, memory consumption, generalization, and robustness to sparse or unstructured data. These architectures set themselves apart from sibling optimization procedures (such as [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) or [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md)) by fundamentally altering the function $f_\Theta$ that defines the radiance field, rather than just the sampling strategy or loss functions used to train it.

## Shared principles
The children in this family share the goal of defining the radiance field function $f: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$, where $\mathbf{x}$ is a 3D position, $\mathbf{d}$ is a viewing direction, $\sigma$ is volume density, and $\mathbf{c}$ is view-dependent color. However, they diverge significantly in how they parameterize this function:
1.  **Decomposition:** Splitting the monolithic MLP into specialized sub-networks (e.g., Sampling vs. Shading in AdaNeRF, or Geometry vs. Appearance in DE-NeRF) [S1, unofficial][S3].
2.  **Representation Change:** Replacing the continuous MLP with discrete structures like voxel grids (SRF, TensoRF, Canonical Radiance Field) or 2D image projections (MultiPlaneNeRF) [S1, unofficial][S4].
3.  **Input/Output Modification:** Altering the inputs (e.g., removing viewing directions in HyperNeRFGAN, using 2D pixel coordinates in R2L) or outputs (e.g., SDF values in RelightingNeRF) [S1, unofficial][S3].
4.  **Hierarchical/Local Modeling:** Using multiple smaller networks for different spatial regions (Mega-NeRF, KiloNeRF) or generating network weights dynamically (HyperNeRFGAN) [S1, unofficial].

## Subtopics

### [AdaNeRF](6195-adanerf.md)
AdaNeRF employs a fixed sample position strategy with incremental sparsity. It splits the NeRF into a lightweight Sampling Network ($\mathcal{N}_s$) and a heavier Shading Network ($\mathcal{N}_c$). The Sampling Network predicts importance weights for fixed samples along a ray in a single evaluation, allowing the Shading Network to process only the top-$k$ significant samples. This reduces the number of samples required for rendering compared to standard NeRF. It is chosen when real-time rendering with fixed sampling is desired.

### [SfMNeRF](8664-sfmnerf.md)
SfMNeRF integrates Structure-from-Motion (SfM) constraints into the NeRF training loop to handle sparse input views. It uses matched SIFT features, epipolar geometry, and multi-view photometric consistency to reduce geometric ambiguity. Unlike standard NeRF which relies on photometric loss alone, SfMNeRF explicitly constrains 3D structure using cross-view geometric priors and a pyramidal optimization strategy. It is chosen for novel view synthesis from sparse inputs where geometric accuracy is critical.

### [DASNeRF](8668-dasnerf.md)
DASNeRF improves sparse reconstruction by integrating a pre-trained depth model (DepthAnythingV2) as a prior. It employs a dynamic optimal sampling mechanism and fuses a five-dimensional vector (coordinates and ray info) into each MLP layer's input. This distinguishes it from AdaNeRF (which lacks explicit depth priors) and SRF (which uses sparse voxels). It is chosen for high-fidelity reconstruction from sparse views using monocular depth priors.

### [R2L (Neural Light Field)](8670-r2l-neural-light-field.md)
R2L is a deep neural light field network that maps 2D pixel coordinates directly to radiance, replacing 3D spatial sampling. It uses a teacher-student distillation approach where a pre-trained NeRF teacher generates pseudo-data for a student network with 88 layers of residual blocks. It trades width for depth under a fixed FLOPs budget to capture fine details. It is chosen for efficient novel view synthesis where camera intrinsics/extrinsics are fixed or handled externally.

### [AligNeRF](8695-alignerf.md)
AligNeRF addresses rendering artifacts from misalignment (moving objects or calibration errors) in high-resolution settings. It combines a convolution-augmented architecture (MLP feature head + 3-layer CNN post-processing) with a staged, alignment-aware fine-tuning procedure. The fine-tuning searches for the best-matching subpatch in ground truth images to correct spatial offsets. It is chosen when input data suffers from motion blur or calibration inaccuracies.

### [HyperNeRFGAN](8765-hypernerfgan.md)
HyperNeRFGAN is a GAN that uses a hypernetwork (StyleGAN2) to generate the weights of a simplified NeRF target network. The target NeRF omits viewing directions and uses Factorized Multiplicative Modulation (FMM) layers. It enables 3D-aware image generation without requiring camera position information during training. It is chosen for generative 3D-aware image synthesis where camera poses are not available or needed.

### [Mega-NeRF Spatial Partitioning](8776-mega-nerf-spatial-partitioning.md)
Mega-NeRF decomposes large-scale scenes into independent spatial cells, each managed by a separate MLP submodule. Query points are assigned to the nearest cell centroid. This allows parallel training and reduces memory consumption by limiting each submodule's data to relevant pixels. It is chosen for city-scale or large-scale environment reconstruction.

### [MultiPlaneNeRF](8875-multiplanenerf.md)
MultiPlaneNeRF replaces trainable neural weights with a non-trainable 2D image-based representation. It projects 3D points onto a fixed set of 2D images to create an aggregate feature vector, decoded by a small shared implicit decoder. This allows a single decoder to generalize across many objects by swapping input 2D images. It is chosen for multi-object scenes requiring fast generalization without per-object training.

### [Sparse Radiance Field (SRF) Representation](8876-sparse-radiance-field-srf-representation.md)
SRF is a voxel-based representation storing density and radiance features in a sparse grid (COO format). It serves as a structured alternative to continuous MLPs, enabling efficient optimization and direct mesh extraction via Marching Cubes. Unlike Mega-NeRF which uses sub-networks, SRF stores explicit feature vectors in voxel coordinates. It is chosen for explicit mesh extraction and efficient sparse volume rendering.

### [KiloNeRF](8942-kilonerf.md)
KiloNeRF uses a teacher-student paradigm where a standard NeRF is distilled into thousands of lightweight MLPs arranged in a 3D grid. It uses a cached sparse 3D grid to skip empty space. It accelerates rendering by replacing the single large MLP with tiny MLPs and spatial acceleration structures. It is chosen for fast rendering of large scenes with complex geometry.

### [Pool proposer architecture](8953-pool-proposer-architecture.md)
The Pool proposer is a trainable component in NeRF-ID that generates fine sampling positions from coarse features. It uses average pooling operations on summed positional encodings and coarse features. It is distinguished from MLPMix by its pooling mechanism, which results in higher variance and worse performance on specific scenes but remains competitive on average. It is chosen as a proposal mechanism for adaptive sampling.

### [Transformer-based Architectures](8998-transformer-based-architectures.md)
Transformer-based architectures apply attention mechanisms to NeRF, such as using Transformers to process multi-view inputs or refine features. They handle global context and long-range dependencies better than MLPs. They are chosen for complex scenes with intricate occlusions or when multi-view consistency is paramount.

### [TensoRF](9034-tensorf.md)
TensoRF models the radiance field as a 4D tensor decomposed into low-rank components (CP or Tucker decomposition). This replaces the reliance on fully connected MLPs, offering faster training and lower space complexity. It is chosen for computational efficiency and scalability in large scenes.

### SinNeRF
SinNeRF uses a depth map to constrain unobserved views via Adversarial loss and Perceptual loss. It is designed for training NeRF on complex scenes from a single image. [S1, unofficial]

### NeRF in the Wild
NeRF in the Wild adds an Appearance embedding to express different lighting conditions and a Transient embedding to model changing geometry. It handles unconstrained photo collections with lighting inconsistencies and dynamic elements. [S1, unofficial]

### StyleNeRF
StyleNeRF is a 3D-aware generative model that merges 3D scene representation with style-based generative models. It uses NeRF to generate low-resolution feature maps and samples them to high resolution to avoid super-computation volume rendering. It applies regularization terms to improve 3D consistency. [S2]

### CLIP-NeRF
CLIP-NeRF adds CLIP (a text-image pre-trained multimodal model) to provide greater freedom in shape operations and global deformation. It uses a disentangled conditional NeRF architecture. Steps include applying a learned deformation network to shape modulation, embedding appearance codes into volume rendering, and allowing control over both shape and appearance. [S2]

### NeRF-RPN
NeRF-RPN uses a multi-scale Object-NeRF architecture. The scene branch encodes geometry and appearance via a voxel representation of 3D neural volume features, and the object branch encodes each independent object. It allows for 3D bounding box regression without rendering. [S2]

### JacobiNeRF
JacobiNeRF synthesizes a NeRF model based on second-order information (Jacobian matrix) to maximize mutual information between highly correlated entities under random scene perturbations. The rendering equation is $\hat{C}(r, z_s, z_c) = \sum_{i=1}^{N_c-1} c_i \alpha_i \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$, where $a_i = 1 - \exp(-\sigma_i \delta_i)$. [S2]

### DE-NeRF
DE-NeRF decomposes lighting, geometry, and appearance to allow relighting of high-frequency environments and appearance editing. It trains a signed distance field (SDF) function using 2D images to reconstruct a display grid. It decomposes the scene into geometric and appearance features (diffuse, roughness, specular). Input geometric features, appearance features, and distance $h$ are fed into MLPs to predict diffuse albedo, roughness, and specular tint. Diffuse reflectance is integrated with the environment map to get diffuse color $c_d$. A specular lighting MLP $F_s$ gets specular lighting $c_l$. Specular color $c_s$ is $c_l$ multiplied by specular hue. Final color $c = c_s + c_d$. On NeRF Synthetic and Shiny Blender datasets, PSNR is 2 points higher than latest papers. [S3]

### RefSR-NeRF
RefSR-NeRF is a reference-guided super-resolution neural radiation field framework combining NeRF and RefSR-CNN. Stage 1 modifies NeRF sampling to patch ray sampling on high-resolution input to get a low-resolution image. RefSR-CNN yields feature maps from detail-dominant and degradation-dominant modules. These are fused to reconstruct a high-resolution image. On the NeRF-LLFF dataset, PSNR is nearly one point higher than the best paper. [S3]

### RelightingNeRF
RelightingNeRF is proposed for handling unstructured photographs and free-viewpoint reconstruction. It changes the model architecture to output SDF values and latent features. The first MLP takes sampling point position and outputs SDF and latent features. Normals, shadow cues, and specular cues are obtained from SDF. The second MLP takes sampling point positions, lighting positions, viewing directions, and latent features to produce the final color. [S3]

### DiffusioNeRF
DiffusioNeRF utilizes a denoising diffusion model to learn priors for scene geometry and color. During NeRF training, random RGBD patches are rendered. The estimated gradients of the log-likelihood are backpropagated to the color and density fields to normalize geometry and color. On the LLFF dataset, PSNR indicators in 3, 6, and 9 input views are 1-2 points higher. Among three input views, PSNR is nearly 1 point higher than ViP-NeRF. [S3]

### GeoNeRF
GeoNeRF is a model with two parts: a geometric morphology reasoner and a renderer. The reasoner builds a cascaded cost volume for source view sequences. The renderer infers geometric and appearance representations using a Transformer-based attention mechanism and cascaded cost volumes, then renders fine images using volume rendering. [S2]

### Canonical Radiance Field with Voxel Grid
Canonical Radiance Field with Voxel Grid is a representation of a static scene at canonical time using a learnable voxel radiance feature grid and a lightweight MLP. It replaces the heavy MLP in NeRF to allow querying finite 3D points. The canonical radiance field is defined as $\mathcal{V}_{\text{R}}^{\text{Can}} = \text{RadNet} (\mathcal{V}_{\text{p}}^{\text{Can}}, \mathcal{V}_{\text{Rf}}^{\text{Can}})$, where $\mathcal{V}_{\text{R}}^{\text{Can}} = \{V\sigma^{\text{Can}}, Vcf^{\text{Can}}\}$ consists of density voxel grid $V\sigma$ and color feature voxel grid $Vcf$. $Vp^{\text{Can}}$ are the 3D coordinates of the voxel grid in canonical space, embedded according to NeRF. $F_{\theta1}$ is a lightweight MLP network used to model the radiance. [S4]

## Background
The canonical NeRF paper [6] introduced the MLP-based implicit representation. Subsequent works have expanded the architecture to address specific limitations. NeRF in the Wild [S1, unofficial] introduced embeddings for lighting and transient objects. Mega-NeRF [S1, unofficial] and KiloNeRF [S1, unofficial] addressed scalability. TensoRF [S1, unofficial] introduced tensor decomposition. SRF [S1, unofficial] introduced sparse voxels. R2L [S1, unofficial] introduced feedforward light fields. AligNeRF [S1, unofficial] addressed misalignment. HyperNeRFGAN [S1, unofficial] combined NeRF with GANs. These developments are documented in various surveys and papers [S1, unofficial][S2, S3, S4].

## Sources
[S1] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
[S2] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
[S3] Methods and Strategies for Improving the Novel View Synthesis Quality of Neural Radiation Field, https://doi.org/10.1109/access.2024.3382997, fetched 2026-10-30
[S4] Forward Flow for Novel View Synthesis of Dynamic Scenes, https://doi.org/10.1109/iccv51070.2023.01468, fetched 2026-10-26
