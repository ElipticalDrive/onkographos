# Specific Model Architectures

Also called: NeRF Architectures, Neural Radiance Field Variants

## Overview
This family encompasses specific neural network topologies and architectural modifications applied to the parametric function $F_\Theta$ that maps 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$ to density $\sigma$ and color $\mathbf{c}$ [Parent]. While the parent article defines the canonical MLP-based implementation, this article details specialized structures designed to address specific limitations such as real-time rendering constraints, single-view ambiguity, geometric accuracy, dynamic scene modeling, and generative capabilities. These architectures differ from standard NeRF by replacing or augmenting the dense MLP with sparse grids, transformers, diffusion priors, or explicit geometric representations to improve efficiency, robustness, or expressiveness [S1, unofficial][S2, S3].

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

### [AdaNeRF](6195-adanerf.md)
AdaNeRF achieves real-time rendering by replacing uniform sampling with a learned importance weighting mechanism. It splits the network into a lightweight Sampling Network and a heavier Shading Network. The Sampling Network predicts weights for fixed sample positions, allowing the Shading Network to process only significant samples. This enables fine-tuning to a compact representation without requiring depth maps.

### [SinNeRF](6199-sinnerf.md)
SinNeRF addresses single-view reconstruction ambiguity by incorporating a perceptual prior via a pre-trained DINO-ViT network and an adversarial discriminator. It constrains unobserved views to prevent the model from collapsing to a flat solution, distinguishing itself from standard NeRF which relies on multi-view consistency.

### [NeRF-RPN](8663-nerf-rpn.md)
NeRF-RPN is a framework for 3D object detection and instance segmentation using a multi-scale Object-NeRF architecture. It predicts 3D axis-aligned bounding boxes and classes directly from the neural field without rendering intermediate views, supporting detection of objects without pre-defined class labels.

### [AligNeRF](8695-alignerf.md)
AligNeRF is an alignment-aware training strategy that combines a convolution-augmented architecture with a staged training procedure. It corrects rendering artifacts caused by misalignment from moving objects or camera calibration errors and improves high-frequency detail recovery in high-resolution settings.

### [MultiPlaneNeRF](8875-multiplanenerf.md)
MultiPlaneNeRF replaces trainable volumetric storage with a non-trainable set of 2D images. It projects 3D points onto these fixed 2D planes to create an aggregate feature vector processed by a small implicit decoder. This allows a single decoder to generalize across many objects by swapping input image sets, solving poor generalization issues.

### [Pool proposer architecture](8953-pool-proposer-architecture.md)
The Pool proposer is a trainable component that generates fine-grained sampling positions from coarse features. It uses average pooling and a bottleneck fully connected layer, making it computationally negligible compared to backbone networks, though it results in higher variance on complex scenes compared to Transformer-based proposers.

### [Transformer-based Architectures](8998-transformer-based-architectures.md)
This family replaces or augments the standard MLP with Transformer architectures to model long-range spatial dependencies. Unlike MLP-based methods that rely on local positional encoding, Transformers utilize self-attention to capture global context, effective for generalizable rendering from sparse views or super-resolution.

### [NeRDi](9112-nerdi.md)
NeRDi leverages general image priors from a pre-trained 2D diffusion model to optimize a NeRF from a single input image. It minimizes diffusion loss under input view constraints, supplemented by semantic and geometric guidance, solving the problem of novel view synthesis without multi-view consistency.

### [Neural Light Field Representations](9273-neural-light-field-representations.md)
This approach replaces iterative volumetric ray-marching with a single forward pass of a deep neural network mapping 4D oriented rays directly to RGB values. It achieves a 26–35× reduction in FLOPs and a 28–31× wall-time speedup compared to NeRF, often with higher PSNR.

### [RING-NeRF](9323-ring-nerf.md)
RING-NeRF introduces a continuous multi-scale scene representation and a decoder latent space invariant to spatial translations and scale changes. It produces intrinsic continuous Level of Detail (LOD) without explicit supervision, distinguishing itself from methods like Zip-NeRF that require specific LOD supervision.

### [Sparse and Efficient Representations](9326-sparse-and-efficient-representations.md)
This family addresses computational bottlenecks by decomposing the scene into localized, discrete, or sparse structures (e.g., voxel grids). It explicitly trades global continuity for spatial sparsity, enabling parallel training and reduced memory footprints compared to global parametric functions.

### [Geometric Priors and Depth Integration](9395-geometric-priors-and-depth-integration.md)
This family improves geometric accuracy under sparse-view conditions by integrating explicit geometric constraints or external depth priors. Unlike standard NeRF, it enforces multi-view geometric consistency or leverages monocular depth estimates to anchor 3D structure.

### [Generative and Style-based Models](9572-generative-and-style-based-models.md)
This family extends NeRF from deterministic reconstruction to probabilistic 3D-aware image synthesis. It replaces photometric losses with generative objectives (adversarial or diffusion) to generate diverse 3D structures from limited data, enabling applications where camera positions are unavailable.

### [IRON](10251-iron.md)
IRON is a neural inverse rendering pipeline that reconstructs high-quality 3D geometry and materials from photometric images captured with a collocated flashlight. It uses a hybrid optimization scheme: volumetric radiance field rendering for topology, followed by physics-based surface rendering for geometry and material disentanglement.

### [pi-GAN](10298-pi-gan.md)
pi-GAN is a generative adversarial approach for unsupervised 3D representation learning. It uses a SIREN backbone with frequency and phase modulation and a StyleGAN-inspired mapping network for conditioning, differing from standard NeRF by using implicit radiance fields conditioned on latent noise.

### [Instance-NeRF](10611-instance-nerf.md)
Instance-NeRF extends a pre-trained NeRF with an additional instance field to detect all objects in a scene, producing bounding boxes and continuous 3D masks. It addresses multi-view inconsistent 2D instance masks without relying on ambiguous 3D bounding volume priors.

### [MIMO-NeRF](10928-mimo-nerf.md)
MIMO-NeRF accelerates inference by replacing the standard SISO MLP with a Multi-Input Multi-Output (MIMO) MLP. It groups spatially neighboring samples to share computation, maintaining original hyperparameters but reducing MLP evaluations per ray, distinguished by a self-supervised training objective.

### [Appearance and Lighting Decomposition](11061-appearance-and-lighting-decomposition.md)
This family explicitly separates scene properties into geometric, material, and lighting components. It reparameterizes the input space (e.g., using surface normals) and outputs distinct physical quantities (albedo, roughness) composed via physically-inspired equations, solving shape-radiance ambiguity.

### [Quadri-Plane Radiance Fields](11128-quadri-plane-radiance-fields.md)
Quadri-Plane Radiance Fields represent a scene using four multi-channel 2D feature planes coupled with a lightweight decoder MLP. This enables computationally efficient volumetric super-resolution by leveraging 2D image super-resolution techniques, separating positional and viewing-direction features into distinct planes.

### [Neural RGB-D Surface Reconstruction](11352-neural-rgb-d-surface-reconstruction.md)
This method generates metric, clean triangle meshes from RGB-D input sequences. It replaces the density-based volumetric representation with a hybrid model: a Truncated Signed Distance Field (TSDF) for geometry and a volumetric radiance field for appearance, optimized jointly with camera poses.

### [DINER](11354-diner.md)
DINER reconstructs a NeRF from a sparse set of RGB views with high relative distances. It leverages a state-of-the-art depth estimator to predict dense depth maps for source views, guiding implicit geometry estimation and improving sampling efficiency compared to methods relying on sparse landmarks.

### [GNeuVox](11357-gneuvox.md)
GNeuVox is a fast-training framework for rendering free-view moving human bodies from monocular video. It integrates neural fields with explicit neural voxel grids (general and individual voxels), enabling training in minutes rather than hours while maintaining comparable quality.

### [HexPlane](11358-hexplane.md)
HexPlane is an explicit representation for dynamic 3D scenes that decomposes a 4D spacetime grid into six feature planes. It computes features by projecting points onto these planes and aggregating them, avoiding large MLP evaluations and achieving over 100× acceleration compared to implicit NeRF methods.

### [STensoRRF](11359-stensorrf.md)
STensoRRF represents a 3D scene using a single 4D tensor decomposed via vector-matrix (VM) decomposition. It decodes appearance features into BRDF values and normal vectors, enabling relighting. It accelerates convergence by 50% compared to existing relighting radiation field models.

### [GIRAFFE](11360-giraffe.md)
GIRAFFE represents scenes as a composition of $N$ distinct entities (objects and background), each with its own neural feature field and learnable affine transformation. This disentangles pose, shape, and appearance, allowing explicit control over individual objects within a generated scene.

### [Ced-NeRF](11364-ced-nerf.md)
Ced-NeRF reconstructs dynamic scenes using a high-resolution voxel grid combined with a canonical space deformation model. It eschews sparse voxel grids or separated static/dynamic fields to solve overfitting artifacts and enable rapid convergence and fast rendering.

### [Dynamic and Deformable Fields](11366-dynamic-and-deformable-fields.md)
This family extends static NeRF to handle dynamic scenes by introducing deformation fields or time-dependent representations. It addresses the challenge of modeling non-rigid motion and topology changes that static radiance fields cannot capture.

#### NeRF in the Wild
- [S1, unofficial] Adds an Appearance embedding to express different lighting conditions.
- [S1, unofficial] Adds a Transient embedding to model changing geometry.

#### CLIP-NeRF
- [S2] A method that adds CLIP (a text-image pre-trained multimodal model) to provide greater freedom in shape operations and global deformation, using a disentangled conditional NeRF architecture.
- [S2] 1. Apply a learned deformation network to shape modulation. 2. Embed appearance codes into volume rendering to map color output. 3. Allow control over both shape and appearance.

#### JacobiNeRF
- [S2] The model synthesizes a NeRF model based on second-order information (Jacobian matrix) to maximize mutual information between highly correlated entities under random scene perturbations. The rendering equation used is: $\hat{C}(r, z_s, z_c) = \sum_{i=1}^{N_c-1} c_i \alpha_i \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$, where $a_i = 1 - \exp(-\sigma_i \delta_i)$ is consistent with discrete volume rendering.

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

#### Edit-NeRF
- [S2] The first method to execute user-controlled deformation models on NeRF, proposing a modular network's conditional radiance field to guide model editing.
- [S2] 1. Train on multiple instances of the same category. 2. Learn underlying part features in an unsupervised manner. 3. Allow user guidance in 2D image shape/texture to propagate to 3D space.
- [S2] Limited capability in shape operations; it only supports adding or deleting local parts of an object. The editing process is slow due to its iterative optimization nature.

#### DT-NeRF
- [S4] Both the Diffusion and Transformer models are trained jointly within a unified optimization framework, sharing the same loss function, and are not pretrained but optimized simultaneously during training through backpropagation.

#### NeRF-A
- [S5] A NeRF variant using appearance embeddings (NeRF-A) without transient embeddings for handling varying camera conditions.
- [S5] Train the model with the left half side of the images in the training set and test novel view synthesis on the right half side.
- [S5] Interpolate between appearance embeddings assigned to brighter and darker images for controllable rendering.
- [S5] Quantitative results on classroom image: PSNR 23.52, SSIM 0.907, LPIPS 0.247.

#### MERF
- [S6] A volumetric scene representation that achieves real-time rendering of large-scale scenes in a browser by combining a sparse feature grid and high-resolution 2D feature planes, with a lossless baking procedure to convert the optimized parameterization into a fast rendering format.
- [S6] MERF uses a unified radiance field representation with two different underlying parameterizations: one for optimization (NGP hash grid) and one for rendering (baked binary occupancy grid and triplanes).
- [S6] During optimization, use the NGP hash grid structure to compress the parameterization, allowing for differentiable sparsification and providing an inductive bias that aids convergence.
- [S6] After optimization, query the recovered NGP to explicitly bake out the MERF and create a binary occupancy grid to accelerate rendering.
- [S6] MERF achieves better image quality for the same amount of compression as SNeRG++, matching the output of a SNeRG++ model that is over 17x larger.
- [S6] Compared to TensoRF, MERF speeds up query time by removing the need for matrix multiplication and halves memory bandwidth consumption.

## Background
Neural Radiance Fields (NeRF) were introduced by Mildenhall et al. in 2020 [S1, unofficial][S2]. The canonical architecture uses dense MLPs, which are computationally expensive and prone to overfitting on small datasets. Subsequent research has focused on optimizing training speed, rendering latency, geometric accuracy, and generalization. Key developments include sparse representations (e.g., Instant NGP, MERF) for efficiency [S1, unofficial][S6], diffusion-based priors (e.g., NeRDi, DiffusioNeRF) for single-view synthesis [S2, S3], and explicit geometric representations (e.g., SDF-based methods like IRON, Neural RGB-D) for better mesh extraction [S1, unofficial]. The field has expanded to include dynamic scenes (HexPlane, GNeuVox) [S1, unofficial], generative models (pi-GAN, GIRAFFE) [S1, unofficial], and specialized tasks like object detection (NeRF-RPN) [S1, unofficial] and super-resolution (RefSR-NeRF) [S3].

## Sources
[S1] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
[S2] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
[S3] Methods and Strategies for Improving the Novel View Synthesis Quality of Neural Radiation Field, https://doi.org/10.1109/access.2024.3382997, fetched 2026-10-30
[S4] DT-NeRF: A Diffusion and Transformer-Based Optimization Approach for Neural Radiance Fields in 3D Reconstruction, https://doi.org/10.62762/tis.2025.874668, fetched 2026-10-26
[S5] HDR-Plenoxels: Self-Calibrating High Dynamic Range Radiance Fields, https://doi.org/10.1007/978-3-031-19824-3_23, fetched 2026-10-26
[S6] MERF: Memory-Efficient Radiance Fields for Real-time View Synthesis in Unbounded Scenes, https://doi.org/10.1145/3592426, fetched 2026-10-26
