# Model Architecture Modifications

Also called: NeRF Architectures, Neural Radiance Field Architectures, Network Structure Variants

## Overview
Model Architecture Modifications define the function approximators (typically Multi-Layer Perceptrons or MLPs) and data structures used to map 3D spatial coordinates and viewing directions to volume density and color. While the original NeRF [S1] uses a single MLP for both properties, subsequent architectures modify this structure to address specific challenges such as inference speed, memory footprint, dynamic scenes, or object manipulation. These variants set themselves apart by altering the input/output mappings, introducing hierarchical or modular components, or replacing dense grids with sparse/latent representations.

## Shared principles
The core task of a NeRF network architecture is to learn a continuous volumetric scene representation $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$, where $\mathbf{x} \in \mathbb{R}^3$ is the spatial position, $\mathbf{d} \in \mathbb{R}^3$ is the viewing direction, $\sigma$ is the volume density, and $\mathbf{c} \in \mathbb{R}^3$ is the view-dependent color. The rendering process integrates these outputs along rays using standard volume rendering equations.

### Standard NeRF Architecture
The baseline architecture [S1] employs two MLPs:
1.  **Position MLP ($\Phi_{pos}$)**: Takes $\mathbf{x}$ and positional encoding $\gamma(\mathbf{x})$ to produce a feature vector $\mathbf{h}$.
2.  **Shading MLP ($\Phi_{shade}$)**: Takes $\mathbf{h}$ and directional encoding $\gamma(\mathbf{d})$ to predict $\sigma$ and $\mathbf{c}$.

## Subtopics

### [DoF-NeRF](9432-dof-nerf.md)
DoF-NeRF is a neural rendering framework that extends Neural Radiance Fields (NeRF) to simulate the depth-of-field (DoF) effects of a physical lens, allowing it to operate on views with different focus configurations and manipulate DoF effects by adjusting virtual aperture and focus parameters. Unlike vanilla NeRF, which assumes a pinhole camera model and all-in-focus inputs, DoF-NeRF explicitly models the aperture and focus distance, enabling it to handle shallow DoF inputs and produce clear, all-in-focus novel view synthesis with less blurriness and more texture details. It achieves this by combining optical models with classical volume rendering to approximate the DoF effect under arbitrary aperture size and focus distance configurations.

### [NeRF-VPT](11770-nerf-vpt.md)
NeRF-VPT is a cascading view prompt tuning paradigm for novel view synthesis that progressively enhances image quality by using RGB information from preceding rendering iterations as visual prompts for subsequent stages. It solves the problem of limited scene understanding in sparse-view or standard NeRF training by iteratively incorporating prior knowledge into the network, distinguishing itself from methods that rely on external networks or ground-truth priors by using only the model's own internal output as guidance.

### [Activation Feature Density Approximation](12492-activation-feature-density-approximation.md)
Activation Feature Density Approximation estimates volumetric density $\hat{d}$ directly from intermediate activation features $f_\ell$ of a coordinate-based MLP, replacing the expensive coarse network evaluation with a lightweight computation. It differs from sibling methods like [Signed Distance Function (SDF) Representations](12495-signed-distance-function-sdf-representations.md) by operating on standard density fields using statistical properties of hidden activations rather than geometric distance metrics, and from [Voxel and Sparse Grid Representations](13738-voxel-and-sparse-grid-representations.md) by remaining fully differentiable and implicit.

### [Signed Distance Function (SDF) Representations](12495-signed-distance-function-sdf-representations.md)
Signed Distance Function (SDF) Representations replace the implicit density field of standard NeRFs with a continuous scalar field representing the distance to the closest surface, providing a more explicit and smooth formulation of geometry. This approach addresses the geometric inaccuracy and surface holes typical of density-based NeRFs by leveraging the zero-level set of the SDF to define geometry [12947-sdf-based-neural-fields.md]. It sets itself apart from siblings like [ProbeSDF](11363-probesdf.md) or [SDF-based Neural Volume Rendering with PBR](15009-sdf-based-neural-volume-rendering-with-pbr.md) by focusing primarily on the geometric representation, whereas those siblings integrate specific appearance models (light field probes or PBR) or architectural modifications (SRDF) on top of the SDF foundation.

### [Volume-Augmented NeRF](12973-volume-augmented-nerf.md)
Volume-Augmented NeRF is a neural rendering method that extends the standard NeRF architecture to handle dynamic fluid scenes by binding explicit fluid volume values to sample points. Unlike original NeRF, which assumes static scenes, this variant allows the renderer to learn density and color based on an explicit fluid distribution provided by a differentiable fluid simulator. It serves as the rendering component in a joint inverse rendering framework for inferring fluid dynamics.

### [Neural-PIL Decomposition Architecture](12975-neural-pil-decomposition-architecture.md)
The Neural-PIL Decomposition Architecture separates scene appearance into geometry, material, and illumination to enable reflectance decomposition under varying lighting conditions. Unlike standard NeRF which predicts view-dependent colors directly, this method predicts density and material parameters (BRDF) in a decomposition network, using a neural pre-integrated lighting (Neural-PIL) model to compute shading at ray termination points. It is distinguished by its two-stage training protocol (pre-training manifolds, then joint optimization) and its shift from volumetric integration to single-point shading at expected ray depths.

### [MINE](13661-mine.md)
MINE is a method for novel view synthesis that models the frustum of the source camera using continuous depth Multiplane Images (MPI), conditioned on input images via a fully-convolutional decoder. It differs from pixelNeRF and GRF, which model the entire 3D space per ray, by reconstructing the source camera's frustum per plane, allowing rendering of any target view via homography warping after a single encoder pass and a fixed number of decoder passes, independent of the number of target views.

### [Voxel and Sparse Grid Representations](13738-voxel-and-sparse-grid-representations.md)
Voxel and Sparse Grid Representations replace the implicit multi-layer perceptron (MLP) coordinate-to-color mapping of standard NeRF with explicit spatial data structures, such as 3D voxel grids, hash tables, or triplanes, to store features directly in space. This approach solves the computational bottleneck of evaluating deep networks for every sample point, enabling real-time rendering and significantly reducing memory usage through sparse reconstruction strategies. Unlike MLP-based approaches that require heavy feed-forward computation for every sample, these methods compute color and density directly from interpolated grid values or lookups.

### [Aleth-NeRF](13764-aleth-nerf.md)
Aleth-NeRF is a NeRF variant that trains on under- and over-exposed images to render normal-light novel views by introducing a 'Concealing Field' assumption, where air particles attenuate emission rather than objects lacking emission. It differs from RAW-NeRF by taking 8-bit LDR images as inputs, whereas RAW-NeRF requires 16-bit HDR RAW images. Unlike methods that apply enhancement networks before or after NeRF training, Aleth-NeRF trains end-to-end without pre-processing or post-processing.

### [Transformer and Vision Model Integration](13911-transformer-and-vision-model-integration.md)
This family integrates Vision Transformers (ViTs) into Neural Radiance Fields to leverage global semantic context, long-range dependencies, and cross-scene transfer capabilities that standard CNN-based encoders lack. It solves the limitations of local feature aggregation by enabling models to reason about occluded regions via self-attention [13878-vit-nerf.md], generalize across scenes without per-scene optimization [13907-transnerf.md], or transfer appearance attributes via semantic correspondence [18133-nerf-analogies.md]. Unlike sibling methods that rely on canonical coordinates or posed multi-view inputs, these approaches utilize transformer attention mechanisms to condition volume rendering on hybrid global-local features or latent scene representations.

### [Human and Specific Subject Modeling](13997-human-and-specific-subject-modeling.md)
This family of methods extends radiance field optimization to dynamic human subjects, solving the problem of high-fidelity novel view and pose synthesis from limited input data (monocular or sparse multi-view). It sets itself apart from general scene modeling by integrating explicit parametric body priors (such as SMPL) to constrain geometry and deformation, thereby preventing overfitting and enabling generalization to unseen poses and subjects.

### [OPONeRF](14014-oponerf.md)
OPONeRF is a neural rendering method that personalizes the parameters of the plenoptic mapping function for each sampled 3D point to handle test-time scene changes and robustness against perturbations such as motion, illumination changes, or contamination. Unlike vanilla NeRFs where network parameters are shared and fixed, or generalizable NeRFs that condition fixed parameters on a latent code, OPONeRF dynamically conditions the renderer's weights on the sampled point's representation and the overall scene geometry. This approach subdivides the scene and discriminatively responds to local variations, providing greater expressiveness and flexibility than methods that model static appearance as stochastic variables.

### [Mixer-NeRF](14073-mixer-nerf.md)
Mixer-NeRF is a 3D reconstruction method that addresses the loss of local information and blurry details in standard Neural Radiance Fields (NeRF) by integrating a hybrid spatial feature learning module (MLCA) and a squeeze-and-excitation (SE) network module. It differs from standard NeRF by explicitly mixing spatial and channel features before the MLP and dynamically weighting features between sampling and inference layers, which improves perceptual image block similarity by more than 30%.

### [Sparse View and Few-Shot Optimization](14428-sparse-view-and-few-shot-optimization.md)
Sparse View and Few-Shot Optimization addresses the degradation of Neural Radiance Fields (NeRF) when trained on a drastically reduced number of input viewpoints, typically $|S| < 20$. In this regime, standard NeRF exhibits divergent behaviors including geometry breakdown, overfitting to input views, and the generation of artifacts that cloud empty space. Unlike methods that rely on large-scale pre-training or external depth priors, this family focuses on in-situ regularization, deformable sampling, and pseudo-observation generation to mitigate overfitting and ambiguity.

### [Mixture-of-Experts and Modular Architectures](14429-mixture-of-experts-and-modular-architectures.md)
Mixture-of-Experts (MoE) and Modular Architectures decompose a single monolithic neural radiance field into multiple specialized sub-networks (experts) coordinated by a gating or routing mechanism. This approach solves the computational bottleneck and representational limitations of standard single-MLP NeRFs by allowing parallel processing of ray samples and specialized handling of complex scene features. Unlike siblings such as [MIMO-NeRF-self-R2](11400-mimo-nerf-self-r2.md) which focus on parallelizing point evaluation within experts, or [Mixture-of-Experts Implicit Neural Representation](13765-mixture-of-experts-implicit-neural-representation.md) which focuses on spatiotemporal compression, this family generally emphasizes dynamic load balancing and expert specialization to improve training speed and rendering quality for large-scale or complex scenes.

### [Uncertainty and Reliability](14487-uncertainty-and-reliability.md)
Uncertainty and Reliability methods extend NeRF architectures to quantify the confidence of rendered outputs, addressing artifacts, redundancy, and data reliability in downstream tasks such as Scene Coordinate Regression (SCR) and botanical phenotyping. Unlike standard deterministic NeRFs, these variants model aleatoric uncertainty (data noise) and epistemic uncertainty (model ignorance) to enable adaptive loss weighting, filtering of poorly rendered images, and distribution-free coverage guarantees via conformal prediction [12968-uncertainty-aware-nerf.md, 13768-ua-plantnerf-architecture.md].

### [Object Decomposition and Composition](14518-object-decomposition-and-composition.md)
Object Decomposition and Composition refers to a family of NeRF architectures that explicitly separate a scene into distinct object instances and background components, enabling instance-level editing, segmentation, and manipulation. Unlike standard NeRFs that treat the scene as a monolithic volume, these methods introduce mechanisms to identify, isolate, and recombine individual objects, allowing for operations such as translation, rotation, and deformation without affecting other scene elements. This family is distinguished from sibling methods like [DoF-NeRF](9432-dof-nerf.md) (which focuses on depth-of-field) or [Semantic and Feature Field Extensions](15556-semantic-and-feature-field-extensions.md) (which adds semantic labels) by its primary goal of structural disentanglement for geometric manipulation and scene editing.

### [Lighting and Appearance Models](14522-lighting-and-appearance-models.md)
Lighting and Appearance Models define how a Neural Radiance Field represents and renders the visual appearance of a scene, specifically addressing view-dependent effects such as specular highlights, subsurface scattering, and stylized textures. While standard NeRF maps 3D coordinates and view directions directly to color via a dense MLP, these models decouple geometry from appearance or replace the implicit color prediction with explicit physical or generative priors . They differ from siblings like [Spherical Harmonics NeRF](14480-spherical-harmonics-nerf.md) by using explicit basis functions for relighting, and from [Neural Pre-Integrated Lighting (Neural-PIL)](13783-neural-pre-integrated-lighting-neural-pil.md) by avoiding pre-integrated light queries in favor of direct coefficient optimization.

### [Optimization and Training Procedures](14565-optimization-and-training-procedures.md)
This family covers the algorithmic procedures, hyperparameter configurations, and optimization loops used to train Neural Radiance Field (NeRF) models and their variants. It addresses the problem of converging implicit 3D representations from 2D observations by defining how model parameters are updated, how sampling strategies interact with the optimizer, and how specific architectural modifications (such as last-layer retraining or activation-based skipping) alter the standard training loop. It sets itself apart from [Model Architecture Modifications](11313-model-architecture-modifications.md) by focusing on the *process* of learning (optimization dynamics, learning rates, batch sizes) rather than the *structure* of the network itself, although the two are tightly coupled.

### [Image Enhancement and Restoration Methods](14566-image-enhancement-and-restoration-methods.md)
Image Enhancement and Restoration Methods address the degradation of Neural Radiance Fields (NeRF) caused by input image artifacts such as blur, transient occlusions, or noise. Unlike standard NeRF training which assumes clean, sharp inputs, these methods explicitly model the degradation process or separate transient elements from the static scene to reconstruct a high-fidelity 3D representation. They differ from sibling methods like [DoF-NeRF](9432-dof-nerf.md) which handles depth-of-field via camera parameters, or [Lighting and Appearance Models](14522-lighting-and-appearance-models.md) which handle illumination, by focusing on spatial or temporal artifacts in the input imagery itself.

### [Modular, Hierarchical, and Latent Architectures](14567-modular-hierarchical-and-latent-architectures.md)
This family of architectures modifies the standard Neural Radiance Field (NeRF) MLP to handle complex scene properties—such as topological changes, hierarchical structures, or non-rigid deformations—that standard continuous coordinate-based networks struggle to represent due to band-limited interpolation artifacts. It sets itself apart from sibling methods like [DoF-NeRF](9432-dof-nerf.md) or [NeRF-VPT](11770-nerf-vpt.md) by explicitly introducing auxiliary latent spaces (ambient dimensions or discrete codes) to decouple structural complexity from the primary radiance field, rather than relying solely on input augmentation or view-dependent pose conditioning.

### [MuRF (Multi-Baseline Radiance Fields)](15007-murf-multi-baseline-radiance-fields.md)
MuRF is a feed-forward method for sparse view synthesis that constructs a 3D volume aligned with the **target** novel view’s frustum, rather than a reference input view. This alignment allows effective aggregation of multi-view information for both small and large baseline settings, where reference-based volumes fail due to insufficient overlap. It differs from ray-based methods (e.g., [IBRNet](13661-mine.md), [MatchNeRF](13661-mine.md)) by using a 3D context-aware CNN decoder to model scene structure explicitly.

### [Gaussian and Primitive-Based Representations](15011-gaussian-and-primitive-based-representations.md)
This family replaces the global implicit coordinate-to-radiance mapping of standard NeRF with explicit, learnable 3D primitives (Gaussians, quadrics, or neural points) to improve rendering efficiency and geometric fidelity. Unlike grid-based or pure MLP approaches, these methods represent the scene as a discrete set of elements, enabling differentiable rasterization or local neighborhood aggregation. This approach is distinct from [Signed Distance Function (SDF) Representations](12495-signed-distance-function-sdf-representations.md) by avoiding signed distance field constraints, and from [Voxel and Sparse Grid Representations](13738-voxel-and-sparse-grid-representations.md) by using continuous or parametric primitives rather than discrete volumetric cells.

### [Decentralized Federated NeRF](15032-decentralized-federated-nerf.md)
DecentNeRF is a decentralized framework for learning global 3D scene representations from crowd-sourced images by distributing training computation to users and aggregating only global radiance field models to minimize server compute and personal data leakage. The core problem solved is achieving photorealistic 3D reconstruction of global scene content while preventing the server from reconstructing transient personal content (e.g., users, objects) present in crowdsourced images taken at different times. It sets itself apart from centralized approaches like NeRF-W by eliminating the need to transfer raw personal images to the server, and from standard federated NeRFs like FedNeRF by explicitly separating personal and global content via dual MLPs and employing secure multi-party computation (SMPC) to prevent server-side reconstruction of personal data.

### [Neural Proxy](15220-neural-proxy.md)
Neural Proxy is a neural rendering method that represents photo-realistic targets as animatable proxies to enable the rendering of unseen animations without temporal learning. It combines neural volume rendering with neural texture techniques, injecting a 3D mesh proxy into the scene to localize particles, thereby decoupling geometry animation from appearance learning. Unlike NeRF, which ties content to continuous Cartesian coordinates making animation difficult, Neural Proxy uses an animatable mesh proxy allowing particles to move with the animation. Unlike Neural Texture, which requires animated training samples to render unseen animations, Neural Proxy renders unseen animations without temporal learning by animating the proxy geometry.

### [ViSNeRF](15255-visnerf.md)
ViSNeRF is a hybrid neural radiance field method that combines explicit feature grids with implicit MLP decoders to synthesize visualizations of dynamic volumetric scenes. It solves the problem of efficiently representing and rendering scenes with arbitrary adjustable parameters (e.g., time, isovalue, transfer functions) by factorizing the scene into a 3D spatial tensor and a K-dimensional parameter tensor. Unlike sibling methods that use spherical harmonics for view-dependent color or fully implicit networks, ViSNeRF uses vector-matrix decomposition for spatial features and CP decomposition for parameters, decoded by small MLPs. This approach reduces memory complexity from $O(N^3 M^K)$ to $O(N^2 M)$ while maintaining high visualization quality.

### [EBAD-NeRF](15441-ebad-nerf.md)
EBAD-NeRF is a framework that jointly optimizes learnable camera poses during exposure time and NeRF parameters to reconstruct sharp 3D representations from blurry images and event data. It explicitly models motion blur via an intensity-change-metric event loss and a photo-metric blur loss. Unlike BAD-NeRF, which uses linear interpolation of start and end poses, EBAD-NeRF uses multiple learnable poses to model complex motion trajectories. Unlike E2 NeRF, which uses pre-deblurred images and COLMAP for fixed poses, EBAD-NeRF jointly optimizes poses and NeRF parameters.

### [Network Architectures](15530-network-architectures.md)
Network architectures in NeRF define the function approximators (typically MLPs) and data structures used to map 3D spatial coordinates and viewing directions to volume density and color. While the original NeRF uses a single MLP for both properties, subsequent architectures modify this structure to address specific challenges such as inference speed, memory footprint, dynamic scenes, or object manipulation. These variants set themselves apart by altering the input/output mappings, introducing hierarchical or modular components, or replacing dense grids with sparse/latent representations.

### [NeurMiPs](15543-neurmips.md)
NeurMiPs is a view synthesis method that represents a 3D scene as a mixture of $K$ learnable rectangular planar surfaces, where each surface acts as an independent "expert" with an associated neural radiance field predicting view-dependent color and transparency. It alleviates the frontal-parallel limitation of traditional Multi-Plane Imagery (MPI) while maintaining high efficiency through closed-form ray-casting, offering better memory and computational efficiency than standard NeRF and superior extrapolation capabilities compared to voxel-based methods like PlenOctrees or KiloNeRF.

### [CodeNeRF](15544-codenerf.md)
CodeNeRF is an implicit 3D neural representation that learns the variation of object shapes and textures across a semantic category, enabling the synthesis of novel views of unseen objects from a single unposed input image. Unlike standard NeRF which is scene-specific, CodeNeRF disentangles shape and texture by learning separate latent embeddings, allowing for joint optimization of camera viewpoint and latent codes at test time. It differs from sibling methods like [DoF-NeRF](9432-dof-nerf.md) and [NeRF-VPT](11770-nerf-vpt.md) by explicitly decoupling the geometry and appearance priors into distinct embedding spaces, whereas [Signed Distance Function (SDF) Representations](12495-signed-distance-function-sdf-representations.md) rely on distance fields rather than volumetric density.

### [Dynamic Scene and Temporal Modeling](15546-dynamic-scene-and-temporal-modeling.md)
Dynamic Scene and Temporal Modeling extends static Neural Radiance Fields (NeRF) to handle non-rigid motion and time-varying scenes, solving the problem of rendering novel views of environments that change over time. Unlike static methods, these approaches introduce temporal dimensions to the radiance field representation, either by warping coordinates to a canonical space, directly conditioning on time, or using explicit deformation fields. This family is distinguished from static NeRF variants by its ability to model temporal dynamics, setting it apart from siblings like [Model Architecture Modifications](11313-model-architecture-modifications.md) which focus on spatial representation improvements without inherent temporal handling.

### [NeRF-Casting](15547-nerf-casting.md)
NeRF-Casting is a method for synthesizing photorealistic specular appearance and reflections in real-world scenes by integrating ray tracing into the Neural Radiance Field (NeRF) rendering pipeline. It improves upon standard NeRF and prior inverse rendering approaches by casting reflection rays from points along camera rays into the NeRF geometry to render feature vectors, which are then decoded into color using a small MLP, rather than querying a large MLP for outgoing radiance. Unlike Ref-NeRF, which reparameterizes outgoing radiance as a function of reflected view direction and requires a large MLP to encode the entire environment, NeRF-Casting uses ray tracing to naturally synthesize consistent reflections of nearby and distant content without requiring the MLP to represent the full environment at every point.

### [RoseNeRF](15548-rosenerf.md)
RoseNeRF is a method for high-fidelity novel view synthesis of real refractive objects that simplifies complex ray interactions inside objects to a single refraction event. It models refraction by predicting a single refracted ray using a 4D light field representation and a refraction network, avoiding the need for Index of Refraction (IoR) fields or detailed surface reconstructions. Unlike deformation field methods that predict position offsets for each sampling point, RoseNeRF predicts the refracted ray and uses hierarchical sampling, avoiding the need for point-by-point deformation and additional priors like IoR.

### [Semantic and Feature Field Extensions](15556-semantic-and-feature-field-extensions.md)
Semantic and Feature Field Extensions generalize the Neural Radiance Field (NeRF) formulation to predict arbitrary per-point feature signals $\mathbf{f}$ in addition to color $\mathbf{c}$ and density $\sigma$ [12995-feature-fields.md]. This allows the representation of semantic labels, physical properties, or latent embeddings within the 3D volume [12995-feature-fields.md]. Unlike standard NeRF which only models appearance, this family lifts 2D observations into a consistent 3D representation, enabling tasks such as zero-shot segmentation, object manipulation, and view-dependent reflection modeling [12995-feature-fields.md, 12404-nerf-for-object-segmentation.md, unofficial].

### [Neural Density-Distance Fields](15665-neural-density-distance-fields.md)
Neural Density-Distance Fields (NeDDF) is a method that represents reciprocally constrained distance and density fields, extending the distance field formulation to adapt to any density field. It solves the limitation of traditional Signed Distance Fields (SDFs) which assume a hard boundary, by allowing the distance field to correspond to various density distributions derived from volume rendering. This enables the representation of boundaryless scenes, transparency, and thin surfaces (thinner than the sampling interval). It sets itself apart from standard NeRF by providing a differentiable distance field that facilitates stable mesh extraction and camera pose localization via reprojection error, while handling the discontinuities of distance fields through an auxiliary gradient in 4D hyperspace.

### [LAENeRF](15738-laenerf.md)
LAENeRF is a unified framework for the photorealistic and non-photorealistic local appearance editing of pre-trained Neural Radiance Fields (NeRFs). It solves the problem of recoloring or stylizing arbitrary 3D regions while minimizing background artifacts and maintaining interactivity. Unlike methods that rely on semantic segmentation masks or single-target colors, LAENeRF learns a palette-based color decomposition mapped from estimated ray terminations, enabling smooth transitions between edited and unedited regions. It distinguishes itself by combining a 3D grid for content selection with region growing for smooth boundaries, and a lightweight NeRF-like module for palette-based color prediction.

### [ExtremeNeRF](15739-extremenerf.md)
ExtremeNeRF is a few-shot view synthesis framework designed to generate novel views from sparse multi-view images captured under unconstrained, varying illumination. It solves the problem of illumination-induced geometry errors in standard NeRF by enforcing multi-view albedo consistency, ensuring that the intrinsic surface color (albedo) remains identical across views regardless of lighting changes. Unlike methods requiring massive image collections (e.g., NeRF-W, RegNeRF) or those that struggle with limited data, ExtremeNeRF uses a two-stage intrinsic decomposition pipeline (FIDNet and PIDNet) to guide NeRF optimization, achieving robust performance with as few as three input views.

### [Conditional and Latent Representations](15989-conditional-and-latent-representations.md)
Conditional and Latent Representations extend the standard NeRF [Parent](3065-nerf-hyperparameters.md) to model an entire object category (e.g., chairs) rather than a single static scene, using instance-specific latent vectors for shape and appearance. This approach solves the problem of efficiently representing diverse 3D instances within a unified model, enabling editing and generation without training separate models per instance. It differs from sibling methods by explicitly decoupling shape and appearance into separate learned codes, often utilizing a shared shape branch or deformation networks to enforce geometric consistency and prevent mutual interference between geometry and color.

### [ActiveNeRF Model Architecture](16203-activenerf-model-architecture.md)
ActiveNeRF extends the standard NeRF architecture by incorporating uncertainty estimation directly into the neural network's output, allowing the model to predict not only volume density and color but also the prediction variance for each sampled point. This modification enables active view selection strategies by quantifying where the model is uncertain, distinguishing it from standard NeRF which outputs only deterministic RGB and density values.

### [SCARF](16591-scarf.md)
SCARF is a method for memory-efficient continual learning of multiple 3D scenes using a single neural network, addressing the catastrophic forgetting problem in NeRF when training on a sequence of scenes with access to only one scene at a time and training data accessible only once. It sets itself apart from siblings like [DoF-NeRF](9432-dof-nerf.md) and [NeRF-VPT](11770-nerf-vpt.md) by factorizing the NeRF MLP into scene-specific and cross-scene shared components generated by a hypernetwork, rather than using prompt tuning or separate density fields. Unlike [Uncertainty and Reliability](14487-uncertainty-and-reliability.md) which focuses on error estimation, SCARF focuses on parameter efficiency and knowledge retention across domain gaps.

### [Multi-input MLP (mi-MLP)](16599-multi-input-mlp-mi-mlp.md)
The multi-input MLP (mi-MLP) is a network architecture modification for Neural Radiance Fields (NeRF) designed to alleviate overfitting in few-shot view synthesis without sacrificing the capacity to synthesize high-frequency details. Unlike vanilla NeRF, which processes inputs through a sequential chain of layers, mi-MLP incorporates the positional encodings of the 3D location and viewing direction into every layer of the network. This structure creates shorter paths between inputs and outputs, allowing the model to learn flexible mappings that prevent overfitting while maintaining the original model capacity.

### [PL-NeRF](18117-pl-nerf.md)
PL-NeRF is a specialized Neural Radiance Field (NeRF) method designed for the 3D reconstruction of power transmission lines (PTLs) in unbounded scenes using progressive motion sequence images. It addresses the specific challenges of thin, low-texture structures—such as discontinuity, jagged edges, and slow rendering—by combining L∞ norm spatial compression, Hash Encoding (HE) for positions, and Integrated Positional Encoding (IPE) for directions. Unlike standard NeRF variants that prioritize real-time rendering speed (e.g., Volinga), PL-NeRF prioritizes reconstruction fidelity and continuity for structured, long-narrow environments.

### [Separate Modeling of Colors and Volume Density](18161-separate-modeling-of-colors-and-volume-density.md)
This method addresses the overfitting and geometry recovery issues in NeRF-based pipelines, particularly in few-shot view synthesis, by decoupling the prediction of volume density and color into two separate Multi-Layer Perceptrons (MLPs). Unlike the standard NeRF architecture which uses a single shared MLP to predict both outputs synchronously from the same inputs, this approach allows for distinct positional encoding frequencies for geometry and appearance. It is designed as a plug-and-play modification to existing NeRF pipelines, prioritizing low-frequency information for density to reflect the assumption that geometry is smoother than appearance.

### NeRF-VAE
- [S1] A generative NeRF model that generalizes to out-of-distribution scenes and removes the need to train on each scene from scratch.
- [S1] The NeRF renderer is conditioned on a latent code. This latent code is trained using Iterative Amortized Inference and a ResNet encoder.
- [S1] An attention-based scene function is introduced instead of the typical MLP.
- [S1] Outperforms baseline NeRF with a low number of scene views (5-20), but is outperformed by baseline NeRF when a large number of views are available (100+) due to lower scene expressivity.

### PIE-NeRF
- [S1] A pipeline integrating physics-based elastodynamic simulation with a neural radiance field representation, using a meshless discretisation based on quadratic generalized moving least squares on top of the density field of a NeRF.
- [S1] Enables large deformations and nonlinear hyperelastic material simulation directly on the implicit representation and supports interactive manipulation of real-world captured scenes with complex geometries.

### DäRF Implementation Details
- [S2, unofficial] NeRF Backbone: K-planes [36].
- [S2, unofficial] MDE Model: DPT-hybrid [38].
- [S2, unofficial] Optimizer: Adam [21].
- [S2, unofficial] NeRF Learning Rate: 1e-2.
- [S2, unofficial] MDE Learning Rate: 1e-5.
- [S2, unofficial] LR Schedule: Cosine warmup.

### NeRF Model Architecture
- [S3, unofficial] The neural network architecture for NeRF, consisting of dense layers with ReLU activations, potentially using view directions as input.
- [S3, unofficial] `D`: The depth of the network (number of hidden layers). Default is 8.
- [S3, unofficial] `input_ch_views`: The number of input channels for view directions. Default is 3.
- [S3, unofficial] `output_ch`: The number of output channels. Default is 4 (RGB + density).
- [S3, unofficial] `use_viewdirs`: Boolean flag to determine if view directions are used as input to the network.

### S3-NeRF
- [S1] A method using shadow and shading cues to infer scene geometry and enable single-image NeRF training.
- [S1] Employ a UNISURF-based occupancy field 3D representation instead of density.
- [S1] Use a modified physics-based rendering equation.
- [S1] Use occupancy-based shadow calculation.

### NeuralEditor
- [S4] A method employing a point cloud-guided NeRF model for shape manipulation, leveraging K-D trees and deterministic integration strategy for accurate point cloud-based rendering, with Phong reflection modeling to enhance color fidelity and geometric detail.
- [S4] 1. Employ point cloud-guided NeRF model. 2. Leverage K-D trees. 3. Use deterministic integration strategy. 4. Achieve accurate point cloud-based rendering. 5. Apply Phong reflection modeling.

### NeRF-Swin
- [S5] A baseline method that performs super-resolution on an LR NeRF using released SwinIR models under the 'Real-World Image Super-Resolution' setting, trained on more than 10k LR-HR pairs.
- [S5] Training set size: More than 10k LR-HR pairs.
- [S5] On synthetic Blender dataset, SwinIR may produce false predictions due to domain mismatch.
- [S5] On real-world LLFF dataset, SwinIR tends to be oversmooth and may predict false color or geometry.
- [S5] Blender ×2 (100x100): PSNR 26.34, SSIM 0.913, LPIPS 0.075.
- [S5] Blender ×4 (100x100): PSNR 24.85, SSIM 0.881, LPIPS 0.108.
- [S5] Blender ×2 (200x200): PSNR 28.03, SSIM 0.926, LPIPS 0.058.
- [S5] Blender ×4 (200x200): PSNR 26.78, SSIM 0.906, LPIPS 0.086.
- [S5] LLFF ×2: PSNR 25.18, SSIM 0.793, LPIPS 0.147.
- [S5] LLFF ×4: PSNR 23.26, SSIM 0.685, LPIPS 0.247.

### Neural Architecture Search for NeRF
- [S6] Train NeRFs with different combinations of network depth and width on a volumetric video frame.
- [S6] Measure the quality of images rendered from the trained NeRFs using Peak Signal-to-Noise Ratio (PSNR).
- [S6] Network Depth ($d$): Number of layers. Initial search space: {1, 2, 4, 6, 8}. Final selected space: {2, 4}.
- [S6] Network Width ($w$): Number of neurons in each layer. Initial search space: {64, 128, 256}. Final selected space: {128, 256}.
- [S6] On RedAndBlack Frame #1450, PSNR was measured for various architectures to determine the optimal search space.
- [S6] Network depth provides the ability to learn hierarchical representations, while network width provides the capacity to 'memorize' the training data. Reducing width monotonically decreases quality, while reducing depth first improves then drops quality if smaller than 2.

## Background
The original NeRF architecture was introduced by Mildenhall et al. [S1] in 2020. It established the baseline of using two MLPs to map positional and directional encodings to density and color. Subsequent research has focused on modifying this architecture to improve performance, efficiency, and applicability to specific domains such as dynamic scenes, sparse views, and generative modeling.

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] DaRF: Boosting Radiance Fields from Sparse Inputs with Monocular Depth Adaptation, https://doi.org/10.48550/arxiv.2305.19201, fetched 2026-10-29 (unofficial)
[S3] YoujiaZhang/NeMF, https://github.com/YoujiaZhang/NeMF, fetched 2026-10-27 (unofficial)
[S4] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
[S5] NeRF-SR: High-Quality Neural Radiance Fields using Supersampling, https://arxiv.org/abs/2112.01759v3, fetched 2026-10-26
[S6] Volumetric Video Compression Through Neural-based Representation, https://doi.org/10.1145/3652212.3652220, fetched 2026-11-05
