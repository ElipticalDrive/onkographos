# Model Architecture Modifications

Also called: NeRF Architecture Variants, Neural Radiance Field Structural Changes

## Overview
Model Architecture Modifications alter the fundamental structure of the Neural Radiance Field (NeRF) representation—specifically the Multi-Layer Perceptrons (MLPs), input conditioning, or output heads—to address limitations in standard NeRF regarding geometry, efficiency, dynamic scenes, or specific physical phenomena [S1, S6, unofficial]. Unlike standard NeRF, which maps 3D coordinates and viewing directions to density and color via a static MLP, these modifications introduce auxiliary inputs (e.g., time, appearance embeddings), auxiliary outputs (e.g., semantics, uncertainty), or substitute the continuous function with discrete structures (e.g., grids, SDFs) [S1, S6, unofficial].

## Shared principles
The standard NeRF architecture [S6, unofficial] consists of two Multi-Layer Perceptrons (MLPs): a coarse network and a fine network. The coarse network takes positional encoded position $\gamma(\mathbf{x})$ and outputs a latent feature vector $\mathbf{h}$ and volume density $\sigma$. The fine network takes $\mathbf{h}$, positional encoded position $\gamma(\mathbf{x})$, and positional encoded view direction $\gamma(\mathbf{d})$ to output color $\mathbf{c}$.

$$
\gamma(\mathbf{x}, d_1, d_2) = [\mathbf{x}, \sin(2^{d_1}\pi\mathbf{x}), \cos(2^{d_1}\pi\mathbf{x}), \dots, \sin(2^{d_2-1}\pi\mathbf{x}), \cos(2^{d_2-1}\pi\mathbf{x})]
$$

Modifications generally fall into three categories:
1.  **Input Expansion:** Adding new conditioning vectors (e.g., appearance embeddings $\mathbf{z}_i$, fluid volume $v$, or previous stage RGB) to the input concatenation.
2.  **Output Expansion:** Adding auxiliary heads to predict values other than $\sigma$ and $\mathbf{c}$, such as semantic labels $y_{aux}$, object codes $o$, or area encoding probabilities $p$. These are rendered using the same transmittance weights $T_i$ and opacity $\alpha_i$ as color:
    $$
    \hat{Y}(r) = \sum_{i=1}^{N} T_i \alpha_i y_{aux}(x_i)
    $$
3.  **Representation Substitution:** Replacing the continuous MLP function with discrete structures (voxel grids, hash tables) or different implicit surfaces (SDFs).

## Subtopics

### [DoF-NeRF](9432-dof-nerf.md)
Extends NeRF to simulate depth-of-field effects by optimizing aperture and focus distance parameters per view, enabling all-in-focus reconstruction from shallow DoF inputs. Choose when input images have varying focus or when explicit lens simulation is required.

### [NeRF-VPT](11770-nerf-vpt.md)
Uses a cascading view prompt tuning paradigm where RGB images from previous rendering stages serve as visual prompts for subsequent networks, improving quality in sparse-view settings. Choose when standard NeRF fails to capture details due to limited input views.

### [MixRT](12401-mixrt.md)
A hybrid representation combining a low-quality mesh, a displacement map, and a compressed NeRF model to enable real-time rendering on edge devices. Choose when real-time performance on constrained hardware is prioritized over pure neural fidelity.

### [Activation Feature Density Approximation](12492-activation-feature-density-approximation.md)
Estimates volumetric density directly from intermediate MLP activation features, replacing expensive coarse network evaluations. Choose when inference speed is critical and coarse sampling can be approximated by feature statistics.

### [Signed Distance Function (SDF) Representations](12495-signed-distance-function-sdf-representations.md)
Models the scene using a Signed Distance Function to provide explicit, smooth surface geometry, addressing the ambiguity of density-based NeRFs. Choose when precise surface extraction is required.

### [SDF-based Neural Fields](12947-sdf-based-neural-fields.md)
Replaces density prediction with SDF values to enable explicit surface modeling via zero-level sets. Choose when geometric accuracy and surface continuity are more important than volumetric appearance.

### [Volume-Augmented NeRF](12973-volume-augmented-nerf.md)
Binds explicit fluid volume values to sample points to handle dynamic fluid scenes. Choose when rendering dynamic fluids where density changes due to flow.

### [Neural-PIL Decomposition Architecture](12975-neural-pil-decomposition-architecture.md)
Separates geometry, material, and illumination into distinct networks to handle varying lighting conditions. Choose when input images have inconsistent illumination and explicit reflectance decomposition is needed.

### [Feature Fields](12995-feature-fields.md)
Generalizes NeRF to predict arbitrary per-point feature signals (e.g., semantics, physical properties) in addition to color and density. Choose when the application requires encoding non-photometric information into the 3D representation.

### [SAID-NeRF](13619-said-nerf.md)
Uses instance masks from Visual Foundation Models to enhance surface estimation for transparent objects by decomposing appearance into diffuse and specular components. Choose when reconstructing transparent objects with ambiguous RGB-only signals.

### [Differentiable Primitive Assembly](13657-differentiable-primitive-assembly.md)
Models 3D shapes as an assembly of quadric primitives using differentiable operations, replacing the continuous density field. Choose when explicit, structured shape abstraction is required.

### [MINE](13661-mine.md)
Models the camera frustum using a continuous depth Multiplane Image (MPI) representation, enabling efficient novel view synthesis via homography warping. Choose when rendering multiple novel views from a single source image efficiently.

### [IE-NeRF](13737-ie-nerf.md)
Separates static scene rendering from transient occlusions using image-dependent 2D mask maps and an inpainting module, avoiding 3D transient fields. Choose when handling "in-the-wild" scenes with transient objects like pedestrians.

### [Voxel and Sparse Grid Representations](13738-voxel-and-sparse-grid-representations.md)
Replaces the continuous MLP with discrete data structures (grids, hash tables) for feature storage, enabling real-time rendering via interpolation. Choose when computational efficiency and real-time performance are primary goals.

### [Aleth-NeRF](13764-aleth-nerf.md)
Trains on under/over-exposed images using "Concealing Fields" to render normal-light views without paired HDR data. Choose when working with standard LDR images captured in adverse lighting conditions.

### [Unified Volumetric Avatar (UVA)](13784-unified-volumetric-avatar-uva.md)
Anchors structured latent codes to a deformable mesh in canonical space for dynamic human avatars, enabling local editing of geometry and texture. Choose when representing dynamic humans with independent geometry/texture editing capabilities.

### [Gear-NeRF](13848-gear-nerf.md)
Stratifies dynamic scenes into motion-based "gears" with region-specific sampling resolution using semantic information. Choose when rendering dynamic scenes with varying motion scales to optimize computational resources.

### [Transformer and Vision Model Integration](13911-transformer-and-vision-model-integration.md)
Integrates Vision Transformers to leverage global semantic understanding and long-range dependencies, addressing the lack of global modeling in standard MLPs. Choose when generalizing across scenes or handling occlusions via attention.

### [Human and Specific Subject Modeling](13997-human-and-specific-subject-modeling.md)
Specialized architectures for representing human subjects or specific object classes, often leveraging canonical spaces or deformation fields. Choose when the target subject has specific structural priors (e.g., human body).

### [OPONeRF](14014-oponerf.md)
Personalizes network parameters for each sampled 3D point to handle test-time scene changes and perturbations. Choose when robustness to local variations (motion, illumination) is required without retraining.

### [Mixer-NeRF](14073-mixer-nerf.md)
Introduces a hybrid spatial feature learning module (MLCA) and SE network to mix spatial and channel features, addressing blurry details in standard NeRF. Choose when high-frequency detail preservation is critical.

### [Lightning NeRF](14134-lightning-nerf.md)
A hybrid representation for large-scale autonomous driving scenes, separating density (voxel) and color (hash grid) with foreground/background decomposition. Choose when reconstructing large-scale outdoor environments with LiDAR data.

### [SCALAR-NeRF](14267-scalar-nerf.md)
A coarse-to-fine framework for large-scale scenes, using a global model and local block models fused via a reference image to avoid "foggy" artifacts. Choose when training large-scale scenes on a single GPU.

### [Viewing Direction Gaussian Splatting](14273-viewing-direction-gaussian-splatting.md)
Combines 3D Gaussian Splatting with a neural network to modulate opacity based on viewing direction, handling specular reflections and transparency. Choose when using Gaussian Splatting but requiring view-dependent appearance effects.

### [Sparse View and Few-Shot Optimization](14428-sparse-view-and-few-shot-optimization.md)
Techniques to mitigate overfitting and geometry breakdown when training with drastically reduced input viewpoints. Choose when input data is extremely sparse ($|S| < 20$).

### [Mixture-of-Experts and Modular Architectures](14429-mixture-of-experts-and-modular-architectures.md)
Replaces the single continuous function with multiple specialized networks (experts) to improve compression and capacity. Choose when a single network struggles with complex features or boundary discontinuities.

### [Uncertainty and Reliability](14487-uncertainty-and-reliability.md)
Extends NeRF to quantify prediction confidence, providing pixel-level reliability metrics for color and depth. Choose when robust filtering and conformal prediction are needed for sparse or unstructured data.

### [Object Decomposition and Composition](14518-object-decomposition-and-composition.md)
Explicitly separates a scene into distinct semantic or geometric object instances for editing and manipulation. Choose when instance-level editing or amodal segmentation is required.

### [Lighting and Appearance Models](14522-lighting-and-appearance-models.md)
Decouples scene geometry from illumination variations using appearance embeddings or pre-integrated lighting queries. Choose when handling real-world image collections with lighting changes and shadows.

### [Optimization and Training Procedures](14565-optimization-and-training-procedures.md)
Algorithmic procedures for optimizing NeRF, including structural modifications to the optimization scope (e.g., skipping empty rays). Choose when managing computational constraints during training.

### [Image Enhancement and Restoration Methods](14566-image-enhancement-and-restoration-methods.md)
Modifies the training pipeline to reconstruct sharp details from blurry or degraded inputs by modeling the degradation process. Choose when input images are low-resolution or defocused.

### [Modular, Hierarchical, and Latent Architectures](14567-modular-hierarchical-and-latent-architectures.md)
Handles non-rigid deformation with topological changes (HyperNeRF) or hierarchical scenes (Strata-NeRF) using latent codes or higher-dimensional spaces. Choose when representing complex deformations or stratified environments.

### [NeRF-W](14630-nerf-w.md)
Extends NeRF to "in-the-wild" photos by disentangling static geometry from transient elements and photometric variations. Choose when reconstructing scenes with changing lighting and moving objects.

### [Generalizable Implicit Reconstruction with SRDF](14760-generalizable-implicit-reconstruction-with-srdf.md)
Uses a Signed Ray Distance Function (SRDF) and ray transformer for generalizable reconstruction with fine details. Choose when avoiding over-smoothing in generalizable implicit reconstruction.

### [TriDF](14761-tridf.md)
A hybrid explicit-implicit representation using triplanes for color and implicit fields for density, accelerating reconstruction from few views. Choose when reconstructing from very few input views (e.g., 3) in remote sensing.

### GARF
- **Definition:** A method using Gaussian activations as an alternative to positional encoding in NeRF, combined with bundle adjustment for pose estimation from unknown camera poses [S1].
- **Steps:** Uses Gaussian activations instead of positional encoding and performs bundle adjustment for pose estimation [S1].
- **Comparison:** Successfully recovers scene representations from unknown camera poses in challenging scenes with low-textured regions [S1].

### NeRF-VAE
- **Definition:** A generative NeRF model that generalizes to out-of-distribution scenes and removes the need to train on each scene from scratch [S1].
- **Steps:** The NeRF renderer is conditioned on a latent code trained using Iterative Amortized Inference and a ResNet encoder. An attention-based scene function is introduced instead of the typical MLP [S1].
- **Comparison:** Outperforms baseline NeRF with a low number of scene views (5-20), but is outperformed by baseline NeRF when a large number of views are available (100+) due to lower scene expressivity [S1].

### PIE-NeRF
- **Definition:** A pipeline integrating physics-based elastodynamic simulation with a neural radiance field representation, using a meshless discretisation based on quadratic generalized moving least squares on top of the density field of a NeRF [S1]. Enables large deformations and nonlinear hyperelastic material simulation directly on the implicit representation [S1].

### DäRF Implementation Details
- **Parameters:** NeRF Backbone: K-planes [S2, unofficial]. MDE Model: DPT-hybrid [S2, unofficial]. Optimizer: Adam [S2, unofficial]. NeRF Learning Rate: 1e-2 [S2, unofficial]. MDE Learning Rate: 1e-5 [S2, unofficial]. LR Schedule: Cosine warmup [S2, unofficial].

### NeRF Model Architecture
- **Definition:** The neural network architecture for NeRF, consisting of dense layers with ReLU activations, potentially using view directions as input [S3, unofficial].
- **Parameters:** `D`: The depth of the network (number of hidden layers). Default is 8 [S3, unofficial]. `input_ch_views`: The number of input channels for view directions. Default is 3 [S3, unofficial]. `output_ch`: The number of output channels. Default is 4 (RGB + density) [S3, unofficial]. `use_viewdirs`: Boolean flag to determine if view directions are used as input to the network [S3, unofficial].

### S3-NeRF
- **Definition:** A method using shadow and shading cues to infer scene geometry and enable single-image NeRF training [S1].
- **Steps:** Employs a UNISURF-based occupancy field 3D representation instead of density, uses a modified physics-based rendering equation, and uses occupancy-based shadow calculation [S1].

### KiloNeRF-style Grid Distillation
- **Definition:** Distills a large teacher NeRF into a spatially partitioned grid of tiny MLPs to enable fast, adaptive evaluation by mapping 3D positions to local student networks [S4, unofficial].
- **Steps:** Initialize a dictionary mapping 3D grid cell indices (i,j,k) to TinyMLP instances. For each training iteration, sample a random ray and generate num_samples stratified samples along it. Compute the grid cell index for each sample by normalizing its position within the bounding box and scaling by grid resolution. Accumulate L2 loss between student and teacher outputs across all samples. Update all grid MLPs using the accumulated loss and the configured learning rate [S4, unofficial].
- **Parameters:** grid_resolution: 3D tuple defining the number of cells per axis, e.g., (16,16,16) [S4, unofficial]. num_distill_iterations: Number of distillation optimization steps, e.g., 1000 [S4, unofficial]. num_samples: Number of stratified samples per ray, e.g., 384 [S4, unofficial]. learning_rate: Optimization step size for grid MLP updates, e.g., 1e-3 [S4, unofficial]. bmin, bmax: Axis-aligned bounding box defining the scene bounds, e.g., [0,0,0] and [1,1,1] [S4, unofficial].

### NeRF with Time Parameter
- **Steps:** Add a time parameter to the model inputs to adapt NeRF for dynamic scenes. Positionally encode the time parameter and concatenate it with the positional encoding of spatial coordinates $\gamma_x(\mathbf{x})$ [S5].
- **Parameters:** Train using the configuration reported for NeRF's real forward-facing experiments [S5].
- **Comparison:** Adding a time parameter improves quality over vanilla NeRF but still lacks detail, suffers from blurry predictions, and fails to reason about scene dynamics [S5].

## Background
The original NeRF architecture by Mildenhall et al. [1] introduced the use of hierarchical rendering with two networks (coarse and fine) to improve computational efficiency. Subsequent works have extensively modified this architecture to address specific challenges such as dynamic scenes [S5], sparse views [S1], generalization [S1], and real-time rendering [S4, unofficial]. The field has evolved from simple MLP modifications to complex hybrid representations involving grids, transformers, and physics-based simulations.

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] DaRF: Boosting Radiance Fields from Sparse Inputs with Monocular Depth Adaptation, https://doi.org/10.48550/arxiv.2305.19201, fetched 2026-10-29 (unofficial)
[S3] YoujiaZhang/NeMF, https://github.com/YoujiaZhang/NeMF, fetched 2026-10-27 (unofficial)
[S4] mdmubarak/OctaNeRF, https://github.com/mdmubarak/OctaNeRF, fetched 2026-10-27 (unofficial)
[S5] Neural Scene Graphs for Dynamic Scenes, https://doi.org/10.1109/cvpr46437.2021.00288, fetched 2026-10-26
