# Specific Model Architectures

Also called: NeRF Architectures, Radiance Field Model Variants

## Overview
Specific Model Architectures refers to the family of neural network structures and representations that extend or modify the canonical Multi-Layer Perceptron (MLP) radiance field to address limitations in training speed, rendering latency, generalization, geometric fidelity, and scene complexity. While the canonical NeRF [Parent](11368-scene-representation-and-architecture.md) uses a single MLP to map coordinates to density and color, this family encompasses methods that replace the global implicit representation with sparse grids, explicit planes, tensor decompositions, or generative priors. These architectures are distinguished from their siblings by their specific inductive biases: for example, [AdaNeRF](6195-adanerf.md) decouples sampling from shading for real-time inference, [MultiPlaneNeRF](8875-multiplanenerf.md) enables zero-shot generalization via 2D image projections, and [HexPlane](11358-hexplane.md) uses explicit 4D feature planes for dynamic scenes.

## Shared principles
The canonical representation maps a 3D position $\mathbf{x} \in \mathbb{R}^3$ and a viewing direction $\mathbf{d} \in \mathbb{R}^3$ to a volume density $\sigma \in \mathbb{R}$ and an RGB color $\mathbf{c} \in \mathbb{R}^3$ using two coupled MLPs: a density network $F_\sigma$ and a color network $F_c$.

**Positional Encoding:**
Raw coordinates are mapped to higher dimensions using positional encoding $\gamma(\mathbf{x})$. For a coordinate $\mathbf{x}$, the encoding includes the original value and sinusoidal functions at increasing frequencies:
$$
\gamma(\mathbf{x}) = (\sin(2^0 \pi \mathbf{x}), \cos(2^0 \pi \mathbf{x}), \dots, \sin(2^{L-1} \pi \mathbf{x}), \cos(2^{L-1} \pi \mathbf{x}))
$$
where $L$ is the number of frequency bands. This allows the MLP to learn high-frequency details [S1, unofficial][S3].

**Network Topology:**
The density network $F_\sigma$ takes the encoded position $\gamma(\mathbf{x})$ as input. It typically consists of $L$ hidden layers with ReLU activations and skip connections from the input to every 4th layer to preserve low-frequency information [S1, unofficial][S3]. The output is the density $\sigma$.

The color network $F_c$ takes the encoded position $\gamma(\mathbf{x})$, the raw viewing direction $\mathbf{d}$, and the intermediate feature vector from the density network (usually the output of the last hidden layer before the density head) as input. It predicts the color $\mathbf{c}$ using a final ReLU or sigmoid activation [S1, unofficial][S3].

**Rendering Integration:**
During training, rays are sampled at points $\mathbf{x}_i$ along the ray. The MLPs predict $\sigma_i$ and $\mathbf{c}_i$. The pixel color $C(\mathbf{r})$ is computed via volumetric rendering:
$$
C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
$$
and $\delta_i$ is the distance between samples [S1, unofficial][S3].

## Subtopics

### [AdaNeRF](6195-adanerf.md)
AdaNeRF is a method that employs fixed sample positions along each ray and incrementally introduces sparsity during training to achieve real-time rendering. It differs from sibling methods like [SinNeRF](6199-sinnerf.md) or [NeRF-RPN](8663-nerf-rpn.md) by decoupling the sampling strategy from the shading network, using a lightweight network to predict weights for all potential points on a line in one pass, thereby reducing the number of points processed by the heavier shading network.

### [SinNeRF](6199-sinnerf.md)
SinNeRF is a method for training Neural Radiance Fields (NeRF) from a single input image, addressing the severe under-constrained nature of single-view reconstruction. It distinguishes itself from standard NeRF by incorporating a discriminator and a perceptual loss based on a frozen DINO-ViT to regularize unobserved views, ensuring the synthesized novel views are perceptually consistent with natural image statistics.

### [NeRF-RPN](8663-nerf-rpn.md)
NeRF-RPN is a method for 3D object detection and instance segmentation that utilizes a multi-scale Object-NeRF architecture to predict 3D bounding boxes and class labels directly from a neural radiance field. It distinguishes itself from standard NeRF variants by enabling the regression of object locations without requiring rendered views for the detection step, and it supports the detection of unlabeled objects as a general framework.

### [AligNeRF](8695-alignerf.md)
AligNeRF is a training strategy and architectural modification for Neural Radiance Fields (NeRF) designed to address misalignment artifacts caused by moving objects or camera calibration errors, while improving high-frequency detail recovery in high-resolution settings. It distinguishes itself from standard NeRF and its siblings by combining a convolution-augmented architecture with a staged, alignment-aware fine-tuning procedure that utilizes patch-based sampling and optical flow estimation to correct spatial misalignments.

### [MultiPlaneNeRF](8875-multiplanenerf.md)
MultiPlaneNeRF is a radiance field representation that replaces the trainable volumetric encoding of vanilla NeRF with a non-trainable 2D image-based representation. It solves the problems of long training times and poor generalization to unseen data by projecting 3D points onto a fixed set of 2D images to create an aggregate feature vector, which is then processed by a small implicit decoder. Unlike vanilla NeRF, which encodes object shape and color in network weights requiring separate training per object, MultiPlaneNeRF allows a single decoder to generalize across many objects by simply swapping the input 2D images without retraining the network.

### [Pool proposer architecture](8953-pool-proposer-architecture.md)
The Pool proposer architecture is a trainable network component used in NeRF-ID to generate fine sampling positions from coarse features along a ray. It solves the problem of adaptive ray sampling by learning to propose high-density regions for volume rendering, improving reconstruction quality over uniform sampling. It is distinguished from its sibling MLPMix proposer by using a bottleneck fully connected layer and sinusoidal positional encodings, whereas MLPMix uses a different mixing strategy; the Pool proposer exhibits higher variance and performs significantly worse on specific scenes like Blender: Materials and Blender: Mic compared to MLPMix.

### [Transformer-based Architectures](8998-transformer-based-architectures.md)
Transformer-based architectures apply self-attention mechanisms to radiance field optimization, enabling the modeling of long-range spatial dependencies and global context that local convolutions often miss. This family is distinct from its siblings by prioritizing global feature aggregation and sequence-based processing over local neighborhood operations, making it particularly effective for generalizable representations and large-scale structure recovery.

### [NeRDi](9112-nerdi.md)
NeRDi is a single-view synthesis model that leverages general image priors from a pre-trained 2D diffusion model to optimize a Neural Radiance Field (NeRF). It solves the problem of novel view synthesis from a single input image by minimizing the diffusion loss of rendered views under input view constraints. Unlike traditional NeRF variants that rely on multi-view consistency or explicit geometric supervision alone, NeRDi uses a pre-trained diffusion model as a regularizer, supplemented by semantic language guides and geometric depth losses to ensure multi-view consistency and 3D structural integrity.

### [Neural Light Field Representations](9273-neural-light-field-representations.md)
Neural Light Field Representations replace the volumetric ray-marching paradigm of NeRF with a direct mapping from camera rays to pixel colors, enabling single-pass inference without iterative sampling. This approach solves the computational bottleneck of volume rendering by treating the scene as a 4D light field, where the input is a ray (origin and direction) and the output is RGB, bypassing the need to query density and radiance at intermediate points along the ray.

### [RING-NeRF](9323-ring-nerf.md)
RING-NeRF (Residual Implicit Neural Grids) is a neural field architecture for 3D scene representation that introduces two primary inductive biases: a continuous multi-scale representation of the scene and an invariance of the decoder’s latent space over spatial and scale domains. It solves the problem of achieving intrinsic, continuous Level of Detail (LOD) without explicit LOD-specific supervision, enabling robust reconstruction across varying observation distances and supporting dynamic resolution extension. Unlike sibling architectures such as [AdaNeRF](6195-adanerf.md) or [SinNeRF](6199-sinnerf.md) which may rely on explicit supervision or fixed hierarchies, RING-NeRF derives LOD from the geometry of the ray-casting cone and the recursive grid structure, allowing it to adapt to unbounded scenes and novel view synthesis with anti-aliasing properties.

### [Sparse and Efficient Representations](9326-sparse-and-efficient-representations.md)
Sparse and Efficient Representations address the computational and memory bottlenecks of dense, global Neural Radiance Fields (NeRFs) by decomposing the scene representation into localized, sparse, or structured components. This family enables scalable training and rendering for large-scale scenes or high-resolution details that would otherwise exceed hardware limits. Unlike global implicit models, these methods partition the scene spatially or structurally, allowing for parallel optimization, reduced query complexity, or adaptive sampling. Key distinctions among siblings include: [Mega-NeRF Spatial Partitioning](8776-mega-nerf-spatial-partitioning.md) uses a 2D grid of independent MLPs for large-scale fly-throughs; [Sparse Radiance Field (SRF) Representation](8876-sparse-radiance-field-srf-representation.md) employs a sparse 3D voxel grid for direct differentiable rendering; [KiloNeRF](8942-kilonerf.md) distills a global teacher into thousands of tiny student MLPs; [ProNeRF](9673-pronerf.md) uses projection-aware sampling to reduce sample counts; and [NeRF-XL Multi-GPU Representation Strategy](#nerf-xl-multi-gpu-representation-strategy) distributes parameters across GPUs for capacity scaling.

### [Geometric Priors and Depth Integration](9395-geometric-priors-and-depth-integration.md)
Geometric Priors and Depth Integration refers to a family of NeRF optimization strategies that incorporate explicit geometric constraints—such as Structure-from-Motion (SfM) correspondences, epipolar geometry, or monocular depth estimates—into the training loop to resolve geometric ambiguity in sparse-view or challenging scenarios. Unlike standard NeRF, which relies solely on implicit volume rendering and photometric consistency, this family enforces structural regularity through auxiliary loss terms, preventing local minima and improving geometry reconstruction fidelity.

### [Generative and Style-based Models](9572-generative-and-style-based-models.md)
Generative and Style-based Models synthesize novel 3D scenes or objects from noise vectors or text prompts by training a Radiance Field as a generator within a Generative Adversarial Network (GAN) framework or by optimizing a NeRF against a pre-trained diffusion model. Unlike inverse rendering methods that reconstruct a specific scene from input images, these methods learn a distribution of 3D shapes and appearances to generate new instances. They are distinguished from sibling methods like [AdaNeRF](6195-adanerf.md) or [SinNeRF](6199-sinnerf.md) by their primary objective of *generation* from latent noise rather than *optimization* of a fixed scene representation, often employing hypernetworks or latent space distillation to bypass the need for dense 3D supervision.

### [IRON](10251-iron.md)
IRON is a neural inverse rendering pipeline that reconstructs high-quality 3D triangle meshes and spatially varying material textures from photometric images captured with a collocated flashlight and moving camera. It distinguishes itself from standard NeRF-based methods by outputting explicit mesh-compatible representations (SDFs and material maps) rather than implicit radiance fields, and from direct mesh optimization methods by employing a hybrid two-stage training scheme that first recovers correct topology via volumetric rendering before refining geometry with edge-aware surface rendering.

### [pi-GAN](10298-pi-gan.md)
pi-GAN is a generative adversarial network (GAN) approach for unsupervised 3D-aware image synthesis that learns a view-consistent 5D spatio-angular radiance field from unlabeled 2D images. It differs from standard 2D GANs by conditioning a SIREN-based implicit radiance field on a latent noise vector to produce volumetric density and color, which are then rendered via volume rendering. Unlike GRAF, which concatenates noise codes, pi-GAN uses a StyleGAN-inspired mapping network with Feature-wise Linear Modulation (FiLM) to condition the SIREN, and it employs a progressive growing strategy to stabilize training and accelerate convergence.

### [Instance-NeRF](10611-instance-nerf.md)
Instance-NeRF is a learning-based 3D instance segmentation pipeline that extends a pre-trained NeRF with an instance field to detect objects, producing bounding boxes, continuous 3D masks, and class labels. It distinguishes itself from semantic NeRF variants by resolving multi-view inconsistency in instance IDs without relying on 3D bounding volume priors, using a 2D mask matching procedure to supervise the continuous instance field.

### [MIMO-NeRF](10928-mimo-nerf.md)
MIMO-NeRF is a neural rendering method that accelerates inference by replacing the standard Single-Input Single-Output (SISO) Multi-Layer Perceptron (MLP) with a Multi-Input Multi-Output (MIMO) MLP. It solves the problem of high computational cost in NeRF by processing groups of $N_p$ neighboring samples on a ray simultaneously, thereby reducing the number of MLP evaluations (# Run) per pixel. Unlike methods that reduce sample counts or network width to save compute, MIMO-NeRF maintains the original NeRF hyperparameters (sample counts and hidden dimensions) but groups inputs to parallelize computation. It distinguishes itself from knowledge distillation approaches by supporting self-supervised training, which avoids the need to train a separate teacher network.

### [Appearance and Lighting Decomposition](11061-appearance-and-lighting-decomposition.md)
Appearance and Lighting Decomposition methods extend standard Neural Radiance Fields by explicitly separating scene geometry, material properties, and lighting conditions. This separation enables high-fidelity relighting of high-frequency environments and independent editing of appearance. Unlike sibling methods that focus on efficient sampling or specific architectural priors (e.g., [AdaNeRF](6195-adanerf.md), [SinNeRF](6199-sinnerf.md)), these models restructure the radiance field to output physical parameters (such as albedo, roughness, or specular coefficients) rather than raw color, allowing for physically plausible synthesis under varying illumination.

### [Quadri-Plane Radiance Fields](11128-quadri-plane-radiance-fields.md)
Quadri-Plane Radiance Fields represent a 3D scene using four multi-channel 2D feature planes coupled with a decoder MLP, designed to enable computationally efficient volumetric super-resolution by leveraging 2D image super-resolution techniques. It diverges from typical 3D coordinate-based schemes by using explicit feature vectors arranged in planes, setting it apart from sibling architectures like [HexPlane](11358-hexplane.md) which uses six planes, or [STensoRRF](11359-stensorrf.md) which uses tensor decompositions.

### [Neural RGB-D Surface Reconstruction](11352-neural-rgb-d-surface-reconstruction.md)
Neural RGB-D Surface Reconstruction is a method for high-quality 3D reconstruction of room-scale scenes from an RGB-D input sequence by training a multi-layer perceptron (MLP) to represent a hybrid scene consisting of a truncated signed distance field (TSDF) for geometry and a volumetric radiance field for appearance. Unlike density-based NeRF variants which often produce noisy, semi-transparent artifacts in free space, this method uses an implicit surface representation to predict hard boundaries between occupied and free space, enabling the extraction of clean, metric triangle meshes via Marching Cubes.

### [DINER](11354-diner.md)
DINER is a method for novel view synthesis that reconstructs a volumetric scene representation (NeRF) from a sparse set of RGB input views by leveraging predicted dense depth maps to guide geometry estimation and sampling efficiency. Unlike standard image-based NeRFs that rely on implicit correspondence or sparse landmarks, DINER uses depth-aware conditioning and depth-guided sampling to handle high relative distances between source cameras, allowing for complete scene capture with fewer input views.

### [GNeuVox](11357-gneuvox.md)
GNeuVox is a fast-training framework for rendering free-view moving human bodies from monocular video by integrating neural fields with explicit neural voxel grids. It solves the high computational cost of traditional NeRF-based human rendering by using two types of voxels: "general voxels" pretrained across multiple bodies to provide geometric priors, and "individual voxels" optimized per subject for unique details. This separation allows training a novel human body in minutes rather than hours, while maintaining rendering quality comparable to methods like HumanNeRF.

### [HexPlane](11358-hexplane.md)
HexPlane is an explicit representation for dynamic 3D scenes that decomposes a 4D spacetime grid into six feature planes spanning each pair of coordinate axes (e.g., XY, ZT). It computes features for points in spacetime by projecting the point onto each feature plane, aggregating the six resulting feature vectors, and passing the fused vector to a tiny MLP to predict color. It sets itself apart from implicit methods (like T-NeRF or deformation-based NeRFs) by avoiding large MLP evaluations and deformation fields, achieving comparable or better synthesis quality with over 100× acceleration. It differs from concurrent works using time-aware MLPs by being a more elegant explicit representation without time-aware MLPs.

### [STensoRRF](11359-stensorrf.md)
STensoRRF is a differentiable volume rendering method that represents a 3D scene using a single 4D tensor to enable high-quality novel view synthesis and relighting. It distinguishes itself from standard NeRF variants by combining tensor decomposition for efficient storage with a direct BRDF estimation network, allowing the scene to be re-illuminated under arbitrary lighting conditions simulated by spherical Gaussians. Unlike methods requiring multiple voxel grids or explicit mesh extraction during training, STensoRRF optimizes a unified tensor field that outputs density, color, normals, and reflectance properties simultaneously.

### [GIRAFFE](11360-giraffe.md)
GIRAFFE represents scenes as a composition of $N$ distinct entities (objects plus a background), where each entity possesses its own neural feature field and an affine transformation. This architecture enables explicit control over the pose, shape, and appearance of individual objects within a scene. It differs from monolithic NeRF representations by disentangling scene elements into separate learnable fields, allowing for generative manipulation of specific entities rather than just the global scene.

### [Ced-NeRF](11364-ced-nerf.md)
Ced-NeRF is a compact and efficient method for modeling dynamic neural radiance fields that utilizes a grid-based hybrid representation and a canonical space to model motion. It specifically addresses the problem of overfitting artifacts, such as floaters and noisy geometry, which commonly occur in hybrid representations of dynamic scenes due to fast convergence before accurate motion prediction is achieved. Unlike methods that use low-resolution grids, sparse voxel grids, or separated fields for static and dynamic components, Ced-NeRF employs a high-resolution grid to ensure rapid convergence while maintaining high-quality rendering results.

### [Dynamic and Deformable Fields](11366-dynamic-and-deformable-fields.md)
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
The canonical NeRF architecture, introduced by Mildenhall et al. in 2020 [S1, unofficial][S3], established the paradigm of implicit neural representations for 3D scenes. Subsequent research rapidly diversified into specific architectures to address NeRF's limitations: high computational cost, lack of generalization to unseen objects, poor performance on dynamic scenes, and sensitivity to sparse or unstructured input data. Early extensions like NeRF in the Wild [S1, unofficial] and NeRF-A [S5] introduced embeddings for appearance and transience. Later works focused on efficiency and explicit representations, such as sparse grids [S1, unofficial][S6], tensor decompositions [S1, unofficial], and 2D plane-based methods [S1, unofficial]. Generative approaches like pi-GAN [S1, unofficial] and NeRDi [S2] integrated diffusion models and GANs, while specialized architectures like HexPlane [S1, unofficial] and Quadri-Plane [S1, unofficial] targeted dynamic and super-resolution tasks. The field continues to evolve with hybrid representations combining implicit and explicit components [S1, unofficial].

## Sources
[S1] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
[S2] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
[S3] Methods and Strategies for Improving the Novel View Synthesis Quality of Neural Radiation Field, https://doi.org/10.1109/access.2024.3382997, fetched 2026-10-30
[S4] DT-NeRF: A Diffusion and Transformer-Based Optimization Approach for Neural Radiance Fields in 3D Reconstruction, https://doi.org/10.62762/tis.2025.874668, fetched 2026-10-26
[S5] HDR-Plenoxels: Self-Calibrating High Dynamic Range Radiance Fields, https://doi.org/10.1007/978-3-031-19824-3_23, fetched 2026-10-26
[S6] MERF: Memory-Efficient Radiance Fields for Real-time View Synthesis in Unbounded Scenes, https://doi.org/10.1145/3592426, fetched 2026-10-26
