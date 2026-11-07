# Specific Model Architectures

Also called: NeRF Architectures, Radiance Field Network Designs

## Overview
Specific Model Architectures refers to the family of neural network designs used within the NeRF training loop to represent the radiance field function $f: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$. While the standard NeRF uses a single, monolithic Multi-Layer Perceptron (MLP), this family encompasses modifications to network depth, width, connectivity, and input/output structures to address limitations in rendering speed, geometric accuracy, generalization, and robustness to misalignment or sparse inputs. These architectures set themselves apart from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) (which focuses on sampling) and [Regularization and Loss Functions](8586-regularization-and-loss-functions.md) (which focuses on optimization objectives) by directly altering the learnable parameters and computational graph of the scene representation itself.

## Shared principles
All methods in this family replace or augment the standard vanilla NeRF MLP. The core mechanism involves defining a function $f_\theta$ parameterized by weights $\theta$ that maps 3D coordinates $\mathbf{x}$ and viewing directions $\mathbf{d}$ to volume density $\sigma$ and color $\mathbf{c}$. The shared mathematical foundation for rendering remains the volume rendering integral:
$$
\mathbf{C}(\mathbf{r}) = \sum_{i=1}^{N} T_i \alpha_i \mathbf{c}_i, \quad \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)
$$
where $\delta_i$ is the distance between samples. The architectural variations primarily affect how $\sigma_i$ and $\mathbf{c}_i$ are computed from the input query $(\mathbf{x}_i, \mathbf{d})$. Common modifications include:
1.  **Decomposition:** Splitting the scene into multiple sub-networks (e.g., spatial partitioning, student-teacher distillation) to reduce per-point computational cost or memory usage [S1, unofficial][S2, S3].
2.  **Input/Output Modification:** Changing the input vector (e.g., removing viewing directions, adding appearance embeddings) or output vector (e.g., predicting features instead of colors, outputting SDF values) to encode specific priors or enable new functionalities [S1, unofficial][S2, S3].
3.  **Structural Augmentation:** Adding convolutional layers, residual blocks, or hypernetworks to the MLP to improve high-frequency detail recovery, training stability, or generative capabilities [S1, unofficial][S2, S3].

## Subtopics

### [AdaNeRF](6195-adanerf.md)
AdaNeRF employs fixed sample positions along each ray and incrementally introduces sparsity during training to achieve real-time rendering. It reduces the number of sample points by training a sampling network to predict all weights for points on a line in one go, thereby reducing the number of sample points required for shading. It sets itself apart from siblings like TermiNeRF by using fixed positions and incremental sparsity rather than depth-conditioned adaptive sampling, and from ENeRF by not requiring multi-view geometry depth prediction.

### [SfMNeRF](8664-sfmnerf.md)
SfMNeRF is a method for novel view synthesis and 3D-scene geometry reconstruction that improves Neural Radiance Fields (NeRF) under sparse input conditions by integrating Structure-from-Motion (SfM) constraints into the training loop. It explicitly constrains the 3D-scene geometry using matched feature positions, epipolar geometry, and multi-view photometric consistency, thereby reducing geometric ambiguity and preventing the model from getting stuck in local minima. Unlike standard NeRF which relies solely on implicit volume rendering losses, SfMNeRF adds explicit geometric priors derived from sparse feature matches and cross-view consistency.

### [DASNeRF](8668-dasnerf.md)
DASNeRF is a Neural Radiance Field (NeRF) optimization method designed for sparse-view 3D reconstruction that improves detail fidelity by integrating monocular depth priors and dynamic sampling. It distinguishes itself from standard NeRF and other sparse-view methods by employing a DepthAnythingV2 model to extract depth priors, which guide the learning process through robust relative depth regularization (Depth Ordering Fidelity Loss and Depth Structural Consistency Regularization), while using a dynamic optimal sampling layer to mitigate lighting artifacts and enhance detail preservation.

### [R2L (Neural Light Field)](8670-r2l-neural-light-field.md)
R2L (R2L) is a deep neural light field network designed for efficient novel view synthesis (NVS) by distilling knowledge from a pre-trained NeRF teacher model. It replaces the standard shallow NeRF MLP with a significantly deeper architecture (depth 88) utilizing extensive residual blocks, trading width for depth under a fixed FLOPs budget to achieve superior rendering quality and 28–31× wall-time speedup. Unlike methods relying on ground-truth depth or discrete voxel grids, R2L learns a continuous radiance field directly from pseudo-data generated by the teacher, allowing it to handle reflective surfaces where depth-based methods fail.

### [AligNeRF](8695-alignerf.md)
AligNeRF is a training strategy and architectural modification for Neural Radiance Fields (NeRF) designed to improve high-frequency detail recovery and robustness to misalignment caused by moving objects or camera calibration errors in high-resolution settings. It distinguishes itself from standard NeRF implementations by combining a convolution-augmented architecture with a staged, alignment-aware fine-tuning procedure that utilizes patch-based sampling and optical flow estimation.

### [HyperNeRFGAN](8765-hypernerfgan.md)
HyperNeRFGAN is a generative adversarial network (GAN) architecture that synthesizes 3D-aware radiance fields by using a hypernetwork to generate the weights of a simplified NeRF target network from a Gaussian noise vector. It solves the problem of 3D-aware image generation when camera positions are unavailable or difficult to estimate, which is common in medical imaging and datasets with limited viewpoints. Unlike standard NeRF-based GANs (e.g., GRAF, π-GAN) that condition a SIREN network on viewing directions, HyperNeRFGAN removes viewing directions from the NeRF input entirely and uses Factorized Multiplicative Modulation (FMM) layers to inject the generated weights, enabling training on unlabeled views or single images per object.

### [Mega-NeRF Spatial Partitioning](8776-mega-nerf-spatial-partitioning.md)
Mega-NeRF Spatial Partitioning decomposes a large-scale 3D scene into a set of spatial cells, each managed by a separate, smaller neural network submodule, to enable parallel training and reduce memory consumption. Unlike monolithic NeRF implementations, this method assigns query points to the nearest cell centroid, allowing the system to scale to city-scale environments by limiting each submodule's training data to only the pixels relevant to its spatial region.

### [MultiPlaneNeRF](8875-multiplanenerf.md)
MultiPlaneNeRF is a radiance field representation that replaces the trainable 3D encoding of vanilla NeRF with a non-trainable 2D image-based representation. It solves the problems of long training times and poor generalization to unseen data inherent in vanilla NeRF by projecting 3D points onto a set of 2D training images to form an aggregate feature vector, which is then processed by a small implicit decoder. Unlike voxel-based methods like SRF or TensoRF that learn spatial structures, MultiPlaneNeRF relies on existing 2D images as the primary representation, allowing the decoder to generalize across many objects by training on a large dataset.

### [Sparse Radiance Field (SRF) Representation](8876-sparse-radiance-field-srf-representation.md)
A Sparse Radiance Field (SRF) is a discrete, voxel-based representation of a 3D scene or object that stores volume density and view-dependent radiance colors in a sparse grid structure. It serves as an efficient alternative to continuous Multi-Layer Perceptrons (MLPs) by encoding the radiance field function $f: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$ into a set of non-empty voxels, enabling fast optimization and direct mesh extraction via Marching Cubes. Unlike dense voxel grids, SRFs utilize a Compressed Sparse Order (COO) format to store only occupied regions, significantly reducing memory usage while maintaining high-fidelity rendering capabilities.

### [KiloNeRF](8942-kilonerf.md)
KiloNeRF is a method that decomposes a large, deep Neural Radiance Field (NeRF) into a set of small, shallow NeRFs arranged in a 3D grid to reduce per-point computational cost. It solves the inefficiency of standard volumetric rendering, where a single monolithic network processes every sample along a ray regardless of local scene complexity. Unlike siblings that use discrete voxel features (e.g., SRF) or hierarchical trees (e.g., PlenOctree), KiloNeRF retains the continuous MLP representation but distributes it spatially, achieving acceleration of up to three orders of magnitude compared to the original NeRF.

### [Pool proposer architecture](8953-pool-proposer-architecture.md)
The Pool proposer is a trainable network component in NeRF-ID that generates fine-grained sampling positions from coarse features along a ray. It solves the problem of adaptive ray sampling by learning to propose additional points where the radiance field likely contains high-frequency details, improving reconstruction quality over uniform or heuristic sampling. It is distinguished from sibling architectures like MLPMix by its use of a bottleneck fully connected layer and sinusoidal positional encoding summation, whereas MLPMix uses a different mixing strategy. Compared to the Transformer proposer, the Pool proposer offers more stable training and better performance on complex scenes, though it exhibits higher variance than MLPMix.

### [Transformer-based Architectures](8998-transformer-based-architectures.md)
Not given in the sources.

#### SinNeRF
Uses Depth map to constrain unobserved views via Adversarial loss and Perceptual loss [S1, unofficial].

#### NeRF in the Wild
Adds an Appearance embedding to express different lighting conditions [S1, unofficial]. Adds a Transient embedding to model changing geometry [S1, unofficial].

#### StyleNeRF
A 3D-aware generative model that merges 3D scene representation with style-based generative models [S2]. Uses NeRF to generate low-resolution feature maps and samples them to high resolution to avoid super-computation volume rendering [S2]. Applies regularization terms to improve 3D consistency [S2].

#### CLIP-NeRF
A method that adds CLIP (a text-image pre-trained multimodal model) to provide greater freedom in shape operations and global deformation, using a disentangled conditional NeRF architecture [S2]. 1. Apply a learned deformation network to shape modulation. 2. Embed appearance codes into volume rendering to map color output. 3. Allow control over both shape and appearance [S2].

#### NeRF-RPN
Uses a multi-scale Object-NeRF architecture where the scene branch encodes geometry and appearance via a voxel representation of 3D neural volume features, and the object branch encodes each independent object [S2].

#### JacobiNeRF
The model synthesizes a NeRF model based on second-order information (Jacobian matrix) to maximize mutual information between highly correlated entities under random scene perturbations. The rendering equation used is: $\hat{C}(r, z_s, z_c) = \sum_{i=1}^{N_c-1} c_i \alpha_i \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$, where $a_i = 1 - \exp(-\sigma_i \delta_i)$ is consistent with discrete volume rendering [S2].

#### DE-NeRF
Decomposes lighting, geometry, and appearance to allow relighting of high-frequency environments and appearance editing [S3]. Train a signed distance field (SDF) function using a set of 2D images to reconstruct a display grid as the input scene for the MLP [S3]. Decompose the scene into geometric features and appearance features (diffuse reflection, roughness, specular tint) [S3]. Input geometric features, appearance features, and distance h (from sampling point to grid) into four corresponding MLPs to predict diffuse albedo, roughness value, and specular tint [S3]. Integrate diffuse reflectance with the environment map to get diffuse color cd [S3]. Train a specular lighting MLP Fs to get specular lighting cl [S3]. Multiply specular lighting cl with specular hue to get specular color cs [S3]. Calculate final color c as cs + cd [S3]. On NeRF Synthetic and Shiny Blender data sets, PSNR is 2 points higher than latest papers [S3].

#### RefSR-NeRF
Reference-guided super-resolution neural radiation field framework combining NeRF and RefSR-CNN to achieve high-quality new perspective synthesis while balancing memory and speed [S3]. Stage 1: Modify NeRF sampling strategy to adopt patch ray sampling on the high-resolution input image to obtain a low-resolution image [S3]. RefSR-CNN yields feature maps from the detail-dominant information module and the degradation-dominant information module [S3]. Fuse the two feature maps to reconstruct a high-resolution image [S3]. On the NeRF-LLFF data set, PSNR is nearly one point higher than the best paper [S3].

#### RelightingNeRF
Proposed for handling unstructured photographs and free-viewpoint reconstruction, changing model architecture to output SDF values and latent features [S3]. Input sampling point position into the first MLP, which outputs SDF values and latent features [S3]. Obtain normals, shadow cues, and specular cues from the SDF values [S3]. Provide sampling point positions, lighting positions, and viewing directions, along with latent features from the first MLP, to the second MLP [S3]. Second MLP produces the final color [S3].

#### DiffusioNeRF
A method that utilizes a denoising diffusion model to learn priors for scene geometry and color, normalizing the geometry and color of the scene to achieve better quality in reconstructed geometry under sparse input views [S3]. During NeRF training, render random RGBD patches. Backpropagate the estimated gradients of the log-likelihood to the color and density fields to normalize the geometry and color [S3]. On the LLFF dataset, the PSNR indicators in 3, 6, and 9 input views are 1-2 points higher. Among three input views, the PSNR index is nearly 1 point higher than ViP-NeRF [S3].

#### GeoNeRF
A model with two parts: a geometric morphology reasoner and a renderer. The reasoner builds a cascaded cost volume for source view sequences. The renderer infers geometric and appearance representations using a Transformer-based attention mechanism and cascaded cost volumes, then renders fine images using volume rendering [S2]. Finally, volume rendering technology is used to render fine images [S2].

#### Latent-NeRF
Applies score distillation to a latent diffusion model and the compact latent space of a pre-trained autoencoder to optimize NeRF [S3]. MLP takes position and direction information as input and outputs volume density and four pseudo color channels in a compact latent space [S3]. Four pseudo color channels correspond to the four latent features used in the diffusion model [S3]. Render a feature map z from any viewpoint of the scene [S3]. Noisily generate a noisy image from the rendered feature map [S3]. Use the stable diffusion model with guidance from text and shape to denoise the image [S3]. Backpropagate the denoising process to the NeRF representation [S3].

#### MS-NeRF
Addresses rendering distortion and blurring caused by reflection or refraction (e.g., mirrors) where standard NeRF fails multi-view consistency. Represents the scene by a set of characteristic fields in parallel subspaces [S3]. Modify NeRF output and volumetric rendering module to output density and features instead of density and color, generating multiple sets of density and features [S3]. Volumetric rendering generates multiple feature maps from the multiple sets of density and features [S3]. Apply a Decoder MLP in each subspace to decode RGB images [S3]. Use a Gate MLP to obtain pixel-level weights for the feature maps [S3]. Integrate features from each subspace by summation to obtain the final result [S3]. On synthetic, real, and RFFR data sets, the PSNR index is 2-4 points higher [S3].

#### NeRDi
Given an input image, use a pre-trained visual language model to introduce two language guides as conditioned reflex input of the diffusion model [S3]. s0 represents the image title containing overall semantics [S3]. s* is the text embedding extracted from the input image, capturing visual clues [S3]. Introduce a geometric loss based on estimated depth mapping to regularize the underlying 3D geometry of NeRF [S3]. On the DTU data set, PSNR is twice as high as traditional NeRF [S3].

#### NeRFLight
Extends grid-based methods by dividing the scene into eight different regions to split NeRF into separate density and color fields, using a shared feature grid [S3]. Avoids increasing the number of features for seamless reconstruction while reducing stored feature values [S3]. Divide the scene into eight different regions, each corresponding to an MLP that outputs density and color [S3]. Use a shared feature grid with features symmetrically placed at the center of the scene [S3].

#### NeRFVS
A method that proposes an overall prior for neural reconstruction to improve NeRF and realize free view synthesis in indoor scenes [S3].

## Background
The original NeRF paper [6] introduced the basic MLP architecture for neural radiance fields. Subsequent research has expanded this into a diverse family of architectures to address specific challenges such as sparse inputs, high-resolution rendering, dynamic scenes, and generative modeling. Key developments include the integration of geometric priors (SfMNeRF, DASNeRF), efficiency improvements through spatial partitioning and distillation (Mega-NeRF, KiloNeRF, R2L), and generative extensions (HyperNeRFGAN, StyleNeRF). The field continues to evolve with the incorporation of pre-trained models (CLIP-NeRF, Latent-NeRF) and advanced sampling strategies (AdaNeRF, Pool proposer).

## Sources
[S1] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
[S2] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
[S3] Methods and Strategies for Improving the Novel View Synthesis Quality of Neural Radiation Field, https://doi.org/10.1109/access.2024.3382997, fetched 2026-10-30
