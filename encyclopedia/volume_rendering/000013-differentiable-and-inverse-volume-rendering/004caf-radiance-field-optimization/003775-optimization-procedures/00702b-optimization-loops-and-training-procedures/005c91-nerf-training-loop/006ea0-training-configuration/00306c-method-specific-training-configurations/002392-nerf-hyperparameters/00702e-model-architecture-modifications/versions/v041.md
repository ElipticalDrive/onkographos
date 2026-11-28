# Model Architecture Modifications

Also called: NeRF Architecture Variants, Implicit Representation Modifications

## Overview
Model Architecture Modifications alter the baseline continuous volumetric representation $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$ [G] to address specific constraints such as dynamic scenes, noise [S1], semantic understanding [S1], or inference speed. These modifications retain the core volume rendering equation but change the input space, network topology, or output fields to enable features like temporal consistency, object manipulation, or noise-aware rendering [S1]. Unlike sibling methods that focus on training hyperparameters or sampling strategies, these modifications fundamentally redefine the neural function or the data structures used to represent the scene.

## Shared principles
The baseline NeRF architecture [Parent](3065-nerf-hyperparameters.md) defines a continuous volumetric scene representation $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$, where $\mathbf{x} \in \mathbb{R}^3$ is the spatial position, $\mathbf{d} \in \mathbb{R}^3$ is the viewing direction, $\sigma$ is the volume density, and $\mathbf{c} \in \mathbb{R}^3$ is the view-dependent color [G]. The rendering process integrates these outputs along rays using standard volume rendering equations [G].

Most modifications in this family retain the core volume rendering equation but alter one or more of the following components:
1.  **Input Space:** Augmenting $\mathbf{x}$ and $\mathbf{d}$ with additional signals such as latent codes $\mathbf{z}$, time $t$, illumination embeddings $\mathbf{z}_l$, or external feature vectors $\mathbf{f}$ from vision models [S1][S2, unofficial][S3, unofficial].
2.  **Network Topology:** Replacing the monolithic MLP with specialized branches (e.g., separate density and color networks), modular experts, or hybrid structures combining MLPs with convolutional or transformer backbones [S1][S2, unofficial].
3.  **Output Space:** Extending the output tuple $(\sigma, \mathbf{c})$ to include additional fields such as uncertainty, semantic labels, or physical properties [S1].

## Subtopics

### [DoF-NeRF](9432-dof-nerf.md)
Extends NeRF to simulate depth-of-field effects by modeling aperture and focus distance, enabling all-in-focus synthesis from defocused inputs.

### [NeRF-VPT](11770-nerf-vpt.md)
A cascading view prompt tuning paradigm that iteratively incorporates rendered images from previous stages as visual prompts to enhance image quality.

### [NeRF-VAE](11991-nerf-vae.md)
A generative model that learns a distribution over 3D scenes via a latent code, enabling zero-shot novel view synthesis without per-scene optimization.

### [Activation Feature Density Approximation](12492-activation-feature-density-approximation.md)
Estimates volumetric density directly from intermediate MLP activation features to replace the coarse density network, targeting inference speedup.

### [Signed Distance Function (SDF) Representations](12495-signed-distance-function-sdf-representations.md)
Replaces implicit volume density with a continuous scalar field representing distance to the nearest surface, providing geometrically accurate surface extraction.

### [Volume-Augmented NeRF](12973-volume-augmented-nerf.md)
Binds explicit fluid volume values to the NeRF input to handle dynamic fluid scenes, serving as the rendering component in an inverse rendering framework.

### [Neural-PIL Decomposition Architecture](12975-neural-pil-decomposition-architecture.md)
Solves reflectance decomposition by separating the scene into a view-dependent appearance model and a physically-based material model, enabling relighting.

### [Feature Fields](12995-feature-fields.md)
Generalizes NeRF to predict arbitrary per-point feature signals $\mathbf{f}$ alongside density and color, enabling tasks like segmentation and detection.

### [SAID-NeRF](13619-said-nerf.md)
Exploits instance masks from Visual Foundation Models to enhance surface estimation for transparent objects by jointly optimizing a semantic field.

### [MINE](13661-mine.md)
Combines Multi-Plane Images (MPI) with NeRF to reconstruct dense 3D representations from single images, with complexity independent of the number of target views.

### [Voxel and Sparse Grid Representations](13738-voxel-and-sparse-grid-representations.md)
Replaces the continuous MLP with explicit spatial data structures (voxels, octrees, hash tables) to enable real-time rendering and scale to large scenes.

### [Aleth-NeRF](13764-aleth-nerf.md)
Synthesizes normal-light views from under/over-exposed inputs by modeling illumination adaptive volume rendering through local and global concealing fields.

### [Transformer and Vision Model Integration](13911-transformer-and-vision-model-integration.md)
Integrates Vision Transformers (ViT) into NeRF architectures to enable generalizable novel view synthesis from single or few images.

### [Human and Specific Subject Modeling](13997-human-and-specific-subject-modeling.md)
Modifies NeRF to represent dynamic, articulated human subjects by integrating parametric body priors and semantic latent codes.

### [OPONeRF](14014-oponerf.md)
Personalizes plenoptic mapping parameters for each sampled 3D point to handle test-time scene changes and robustness.

### [Mixer-NeRF](14073-mixer-nerf.md)
Addresses loss of local detail by introducing a hybrid spatial feature learning module (MLCA) and Squeeze-and-Excitation (SE) module to mix spatial and channel features.

### [Sparse View and Few-Shot Optimization](14428-sparse-view-and-few-shot-optimization.md)
Addresses overfitting and artifact generation in sparse-view settings by introducing in-situ depth priors or pseudo-observations.

### [Mixture-of-Experts and Modular Architectures](14429-mixture-of-experts-and-modular-architectures.md)
Decomposes the scene representation into specialized sub-networks (experts) dynamically activated based on input characteristics to reduce computation.

### [Uncertainty and Reliability](14487-uncertainty-and-reliability.md)
Extends NeRF architectures to quantify aleatoric and epistemic uncertainty, enabling filtering of poorly rendered views and reliable geometry assessment.

### [Object Decomposition and Composition](14518-object-decomposition-and-composition.md)
Decomposes the scene representation into distinct components (foreground/background, object instances) for instance-level editing and segmentation.

### [Lighting and Appearance Models](14522-lighting-and-appearance-models.md)
Decouples intrinsic scene properties (geometry, albedo) from extrinsic appearance factors (illumination, style) for explicit lighting control.

### [Image Enhancement and Restoration Methods](14566-image-enhancement-and-restoration-methods.md)
Handles input image defects (blur, occlusions) by modeling the degradation process or transient elements as part of the optimization.

### [Modular, Hierarchical, and Latent Architectures](14567-modular-hierarchical-and-latent-architectures.md)
Introduces modularity and hierarchical structures to handle non-rigid deformation, topological changes, and multi-level stratification.

### [NeuralEditor](14895-neuraleditor.md)
Enables scene editing by exploiting an explicit point cloud representation as the structural basis for the implicit field.

### [MuRF (Multi-Baseline Radiance Fields)](15007-murf-multi-baseline-radiance-fields.md)
A feed-forward method for sparse view synthesis that constructs a target view frustum volume aligned with the novel view.

### [Gaussian and Primitive-Based Representations](15011-gaussian-and-primitive-based-representations.md)
Replaces the global continuous volumetric field with explicit, differentiable geometric primitives (Gaussians, points) for speed and interpretability.

### [Decentralized Federated NeRF](15032-decentralized-federated-nerf.md)
A decentralized framework for learning global 3D scene representations from crowd-sourced images while preserving privacy.

### [Neural Proxy](15220-neural-proxy.md)
Represents photo-realistic targets as animatable proxies (meshes with neural textures) to enable rendering of unseen animations.

### [ViSNeRF](15255-visnerf.md)
Efficiently synthesizes visualizations from dynamic volumetric scenes by combining explicit spatial feature grids with implicit MLP decoders using tensor decomposition.

### [EBAD-NeRF](15441-ebad-nerf.md)
Reconstructs sharp 3D radiance fields from motion-blurred images and hybrid event-RGB data by modeling complex camera motion blur.

### [DINO-enhanced NeRF](15497-dino-enhanced-nerf.md)
Integrates pre-trained DINO vision features into the NeRF architecture to enhance 3D reconstruction, particularly in few-shot scenarios.

### [NeurMiPs](15543-neurmips.md)
Represents a scene as a mixture of learnable rectangular planar surfaces, each acting as an independent expert predicting view-dependent color and transparency.

### [CodeNeRF](15544-codenerf.md)
An implicit 3D representation that learns variation of object shapes and textures across a semantic category, enabling synthesis from a single unposed image.

### [Dynamic Scene and Temporal Modeling](15546-dynamic-scene-and-temporal-modeling.md)
Extends static NeRF to represent scenes changing over time by augmenting input space with temporal signals or deformation fields.

### [NeRF-Casting](15547-nerf-casting.md)
Synthesizes specular reflections by replacing neural network queries for outgoing radiance with ray tracing into the NeRF geometry.

### [RoseNeRF](15548-rosenerf.md)
High-fidelity synthesis of real refractive objects by simplifying complex ray interactions to a single refraction event using a 4D light field representation.

### [NeRF-Det](15549-nerf-det.md)
Indoor 3D object detection that explicitly models scene geometry as an opacity field, enabling gradient flow from NeRF to the detection pipeline.

### [Semantic Ray (S-Ray)](15552-semantic-ray-s-ray.md)
Learns a generalizable semantic field that infers semantic labels on novel viewpoints across unseen scenes using cross-reprojection attention.

### [Neural Density-Distance Fields](15665-neural-density-distance-fields.md)
Reciprocally constrains distance and density fields, deriving density from the first derivative of an output distance field.

### [LAENeRF](15738-laenerf.md)
A unified framework for photorealistic and non-photorealistic local appearance editing of pre-trained NeRFs using a 3D edit grid and palette-based neural module.

### [ExtremeNeRF](15739-extremenerf.md)
Few-shot view synthesis under unconstrained illumination by enforcing multi-view albedo consistency via intrinsic decomposition.

### [Conditional and Latent Representations](15989-conditional-and-latent-representations.md)
Introduces low-dimensional latent codes $\mathbf{z}$ to condition the radiance field, enabling generative modeling of object categories.

### [ActiveNeRF Model Architecture](16203-activenerf-model-architecture.md)
Predicts prediction variance (uncertainty) for each sampled point to facilitate active view selection strategies.

### [SCARF](16591-scarf.md)
Memory-efficient continual learning of multiple 3D scenes using a single neural network by factorizing MLP weights into scene-specific and shared components.

### [PL-NeRF](18117-pl-nerf.md)
Specialized NeRF for power transmission lines, using $L_\infty$ norm spatial compression and Integrated Positional Encoding to reduce aliasing.

### [Separate Modeling of Colors and Volume Density](18161-separate-modeling-of-colors-and-volume-density.md)
Decouples density and color prediction into two distinct networks with different positional encoding frequencies to address overfitting.

### [Network Structure Modifications](20497-network-structure-modifications.md)
General category for altering input space, topology, or output fields to address dynamic scenes, noise, or semantic understanding.

### [Neural blend weight field network](21760-neural-blend-weight-field-network.md)
Predicts residual blend weights for skeletal animation by combining spatial positional encoding with per-frame latent codes.

### PIE-NeRF
PIE-NeRF is a pipeline integrating physics-based elastodynamic simulation with a neural radiance field representation [S1]. Instead of converting the learned implicit surface into a mesh or voxel grid, the method uses a meshless discretisation based on quadratic generalized moving least squares on top of the density field of a NeRF [S1]. This enables large deformations and nonlinear hyperelastic material simulation directly on the implicit representation and supports interactive manipulation of real-world captured scenes with complex geometries [S1].

### DäRF Implementation Details
DäRF is implemented based on K-planes [36] as NeRF [S2, unofficial]. The MDE Model is DPT-hybrid [38] [S2, unofficial]. The Optimizer is Adam [21] [S2, unofficial]. The NeRF Learning Rate is 1e-2 [S2, unofficial]. The MDE Learning Rate is 1e-5 [S2, unofficial]. The LR Schedule is Cosine warmup [S2, unofficial].

### S3-NeRF
S3-NeRF is a method using shadow and shading cues to infer scene geometry and enable single-image NeRF training [S1]. It employs a UNISURF-based occupancy field 3D representation instead of density [S1]. It uses a modified physics-based rendering equation [S1]. It uses occupancy-based shadow calculation as key implementation differences [S1].

### NeRF with Frozen DINO Features
NeRF with Frozen DINO Features is a NeRF variant that integrates pre-trained DINOv2 features by projecting them to match NeRF input dimensions and concatenating them with positional encodings before feeding into the NeRF MLP [S3, unofficial]. The feature projection dimension is 256 (matching NeRF hidden units) [S3, unofficial]. The DINO Model is DINOv2-base, pre-trained on ImageNet, frozen during training [S3, unofficial]. Feature Extraction involves the DINOv2-base model outputting dense feature maps $F_i \in \mathbb{R}^{H \times W \times D}$ where $D=768$ [S3, unofficial]. Features are bilinearly sampled at projected 2D coordinates $(u_i, v_i) = \Pi(P_i, x)$ for each 3D point $x$ [S3, unofficial]. The formula for projection is $f'(x) = W_f \cdot f(x) + b_f$ where $f(x)$ is the aggregated feature vector from all views, and $W_f, b_f$ are learned linear transformation parameters [S3, unofficial]. The input to the NeRF MLP is $F_\theta([\gamma(x), \gamma(d), f'(x)]) \rightarrow (c, \sigma)$ where $f'(x)$ is projected to 256 dimensions [S3, unofficial].

### NeRF-Swin
NeRF-Swin is a baseline method that performs super-resolution on an LR NeRF using released SwinIR models under the 'Real-World Image Super-Resolution' setting, trained on more than 10k LR-HR pairs [S4]. On synthetic Blender dataset, SwinIR may produce false predictions due to domain mismatch [S4]. On real-world LLFF dataset, SwinIR tends to be oversmooth and may predict false color or geometry [S4]. Metrics: Blender ×2 (100x100): PSNR 26.34, SSIM 0.913, LPIPS 0.075 [S4]. Blender ×4 (100x100): PSNR 24.85, SSIM 0.881, LPIPS 0.108 [S4]. Blender ×2 (200x200): PSNR 28.03, SSIM 0.926, LPIPS 0.058 [S4]. Blender ×4 (200x200): PSNR 26.78, SSIM 0.906, LPIPS 0.086 [S4]. LLFF ×2: PSNR 25.18, SSIM 0.793, LPIPS 0.147 [S4]. LLFF ×4: PSNR 23.26, SSIM 0.685, LPIPS 0.247 [S4].

### Neural Architecture Search for NeRF
Neural Architecture Search for NeRF trains NeRFs with different combinations of network depth and width on a volumetric video frame [S5]. It measures the quality of images rendered from the trained NeRFs using Peak Signal-to-Noise Ratio (PSNR) [S5]. Network Depth ($d$) is the number of layers, with an initial search space of {1, 2, 4, 6, 8} and a final selected space of {2, 4} [S5]. Network Width ($w$) is the number of neurons in each layer, with an initial search space of {64, 128, 256} and a final selected space of {128, 256} [S5]. On RedAndBlack Frame #1450, PSNR was measured for various architectures to determine the optimal search space [S5]. Network depth provides the ability to learn hierarchical representations, while network width provides the capacity to 'memorize' the training data [S5]. Reducing width monotonically decreases quality, while reducing depth first improves then drops quality if smaller than 2 [S5].

### Deformable NeRF
Not given in the sources.

### Multi-Identity NeRF
Multi-Identity NeRF (MI-NeRF) is a neural rendering method that learns a single network to represent multiple identities by modeling multiplicative interactions between facial expressions and identity codes, enabling expression transfer and lip-syncing across unseen identities [S6, unofficial]. It learns a multiplicative module M(E_i, i) that conditions the network, where E_i is the expression code for image i and i is the learnable identity code [S6, unofficial]. The Identity code dimension d is set to 16 [S6, unofficial]. Unlike single-identity methods (AD-NeRF, LipNeRF) that learn audio-lip representations from only 3 or 20 seconds, MI-NeRF leverages information from multiple identities for more accurate lip synchronization [S6, unofficial]. It is used for facial expression transfer and lip-synced video synthesis [S6, unofficial].

### IntrinsicNeRF Model Architecture
IntrinsicNeRF is a model that outputs reflectance, shading, and residual terms for real-time augmented applications [S7]. It is trained using unsupervised prior and reflectance clustering as constraints in the loss function [S7].

### Piecewise Linear NeRF
Piecewise Linear NeRF (PL-NeRF) is a NeRF variant that uses a piecewise linear opacity formulation for volume rendering, replacing the standard piecewise constant opacity assumption [S8]. It optimizes separate networks for coarse and fine models, jointly trained with MSE loss on ground truth images [S8]. Batch size is 1024 rays [S8]. Learning rate is $5 \times 10^{-4}$, decaying exponentially to $5 \times 10^{-5}$ [S8]. Training iterations are 500k per scene [S8]. Coarse samples are 128 [S8]. Fine samples are 64 [S8]. Hardware is a Single Nvidia V100 GPU [S8]. It outperforms vanilla NeRF (constant opacity) in all metrics (PSNR, SSIM, LPIPS) on Blender and LLFF datasets [S8]. It is implemented on top of NeRF-Pytorch [38] by changing volume rendering to piecewise linear opacity formulation and utilizing exact importance sampling [S8].

## Background
The original NeRF implementation by Mildenhall et al. [1] established the baseline continuous volumetric representation. Subsequent works have extensively modified this architecture to address limitations in speed, generalization, dynamic scenes, and physical accuracy. Reviews such as [S1] categorize these modifications into improvements for training/inference speed, handling of dynamic scenes, and integration with other vision tasks. Specific variants like PIE-NeRF [S1] integrate physics-based simulation, while others like S3-NeRF [S1] focus on single-image training using occupancy fields. The integration of pre-trained vision models, such as DINO [S3, unofficial], and super-resolution techniques like NeRF-Swin [S4], represents a shift towards leveraging external priors. Architectural searches [S5] have identified optimal network depths and widths for volumetric video. Methods like MI-NeRF [S6, unofficial] and IntrinsicNeRF [S7] extend the output space for specific applications like lip-syncing and augmented reality. Finally, PL-NeRF [S8] addresses quadrature instability in volume rendering.

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] DaRF: Boosting Radiance Fields from Sparse Inputs with Monocular Depth Adaptation, https://doi.org/10.48550/arxiv.2305.19201, fetched 2026-10-29 (unofficial)
[S3] Limitations of NERF with pre-trained Vision Features for Few-Shot 3D Reconstruction, https://arxiv.org/abs/2506.18208v1, fetched 2026-10-26 (unofficial)
[S4] NeRF-SR: High-Quality Neural Radiance Fields using Supersampling, https://arxiv.org/abs/2112.01759v3, fetched 2026-10-26
[S5] Volumetric Video Compression Through Neural-based Representation, https://doi.org/10.1145/3652212.3652220, fetched 2026-11-05
[S6] MI-NeRF: Learning a Single Face NeRF from Multiple Identities, https://arxiv.org/abs/2403.19920v2, fetched 2026-10-26 (unofficial)
[S7] Semantically-aware Neural Radiance Fields for Visual Scene Understanding: A Comprehensive Review, https://link.springer.com/article/10.1007/s11263-025-02663-5, fetched 2026-10-27
[S8] NeRF Revisited: Fixing Quadrature Instability in Volume Rendering, https://doi.org/10.52202/075280-1139, fetched 2026-10-26
