# Model Architecture and Encoding

Also called: Compact NeRF Architecture & Ray Casting, Mildenhall et al. (2021) NeRF, NeRF Model Architecture and Encoding, Wang et al. (2021) NeRF

## Overview
This article defines the canonical neural radiance field (NeRF) architecture and its associated positional encoding scheme, which map 3D spatial coordinates and viewing directions to volume density and RGB color [S1]. It solves the problem of learning continuous, view-dependent appearance and geometry from discrete 2D images by overcoming the spectral bias of standard Multi-Layer Perceptrons (MLPs) through high-frequency sinusoidal feature mapping [S1]. What sets this baseline apart from its siblings is its specific configuration of fully-connected layers, skip connections, and the hierarchical positional encoding function that serves as the foundational template for most subsequent NeRF variants [S1].

## Shared principles
The core mechanism involves transforming low-dimensional input coordinates into a high-dimensional feature space using sinusoidal positional encoding, which is then processed by a fully-connected MLP to predict radiance field properties [S1].

**Positional Encoding**
The input coordinates $\mathbf{x} \in \mathbb{R}^3$ (position) and $\mathbf{d} \in \mathbb{R}^3$ (view direction) are mapped to higher dimensions using a hierarchical positional encoding function $\gamma(\cdot)$ [S1]. For a scalar input $p$, the encoding is defined as:
$$ \gamma(p) = (\sin(2^0 \pi p), \cos(2^0 \pi p), \dots, \sin(2^{L-1} \pi p), \cos(2^{L-1} \pi p)) $$
where $L$ is the number of frequency bands [S1]. The encoded vector has dimension $2L$. For a 3D position $\mathbf{x}$, the encoding is applied component-wise, resulting in $3(2L_x + 1)$ features if the original coordinate is included, or $6L_x$ if not. In the canonical implementation, the input position features are $3 \times (2^{L_x} + 1) = 63$ features, implying $L_x=10$ and inclusion of the original coordinate [S1]. Similarly, input direction features are $3 \times (2^{L_d} + 1) = 27$ features, implying $L_d=4$ [S1].

**Network Architecture**
The network consists of two branches sharing a common positional encoding input for position, and a separate encoding for direction:

1.  **Density Branch:** Takes the encoded position features (63 dimensions). It consists of four fully connected (FC) layers with 192 hidden units each, followed by ReLU activations. The final layer maps 192 units to a single scalar output representing volume density $\sigma$ [S1].
    *   Layer 1: FC 63→192, ReLU
    *   Layer 2: FC 192→192, ReLU
    *   Layer 3: FC 192→192, ReLU
    *   Layer 4: FC 192→192, ReLU
    *   Output: $\sigma \in \mathbb{R}$

2.  **View-Dependent Color Branch:**
    *   **Feature Extraction:** The output of the fourth FC layer in the Density Branch (192 units) is passed through an additional FC layer to produce a 128-dimensional view-independent feature vector $f$ [S1].
    *   **Direction Encoding:** The viewing direction $\mathbf{d}$ is encoded using the directional positional encoding (27 dimensions) [S1].
    *   **Color Prediction:** The view-independent feature $f$ (128 units) is concatenated with the encoded direction (27 units) to form a 155-dimensional vector. This vector is passed through two FC layers with 64 hidden units each, followed by ReLU activations. The final layer maps 64 units to 3 units representing RGB color [S1].
        *   Layer 1: FC 155→64, ReLU
        *   Layer 2: FC 64→3 (RGB)

**Ray Sampling**
Rays are parameterized as $r(\lambda) = o_{world} + \lambda d_{world}$, where $o_{world}$ is the camera origin and $d_{world}$ is the world-space direction derived from camera intrinsics and extrinsics [S1]. For each ray, $N_{samples} = 128$ stratified samples are drawn from the near and far planes, typically normalized to $[0, 1]$ in NDC space [S1].

## Subtopics

### [SUDS](14964-suds.md)
SUDS is a 4D radiance field representation designed for large-scale dynamic urban scenes, enabling free-viewpoint rendering, semantic decomposition, and 3D scene flow estimation across thousands of videos and millions of frames. It distinguishes itself from prior methods by factorizing the scene into three separate multi-resolution hash table branches (static, dynamic, and far-field) and utilizing unlabeled supervision signals (RGB, sparse LiDAR, optical flow, and DINO descriptors) rather than requiring expensive 3D bounding box or panoptic segmentation annotations.

### [Rip-NeRF](15532-rip-nerf.md)
Rip-NeRF is a hybrid radiance field method that renders high-fidelity, anti-aliased images by combining cone casting with an explicit representation of anisotropic 3D Gaussians. It distinguishes itself from sibling methods like Zip-NeRF by using **Ripmap Encoding** and **Platonic Solid Projection** to efficiently featurize these Gaussians, avoiding the computational inefficiency of multi-sampling while maintaining compact storage.

### [UE4-NeRF](15539-ue4-nerf.md)
UE4-NeRF is a neural rendering system designed for the real-time interactive rendering of large-scale 3D scenes by integrating Neural Radiance Fields (NeRF) with the Unreal Engine 4 (UE4) rasterization pipeline. It partitions large environments into sub-NeRFs, representing each block using textured polygonal meshes initialized from regular octahedra, and utilizes a Level of Detail (LOD) hierarchy to maintain high frame rates at varying observation heights. Unlike Block-NeRF, which is limited to grid structures, UE4-NeRF supports diverse terrains including mountainous areas, and unlike standard NeRF, it decouples geometry and appearance into mesh and shader components.

### [Multi-view Aggregation with Skeletal Features](16433-multi-view-aggregation-with-skeletal-features.md)
This method integrates geometric priors from a parametric human model (SMPL) with multi-view image features to improve the quality of neural radiance fields for human performance rendering. It solves the problem of ambiguous geometry and texture in single-view or poorly constrained multi-view setups by using skeletal features as a robust, time-augmented geometric backbone. Unlike standard NeRF which relies solely on positional encoding of 3D coordinates, this approach explicitly retrieves features from a learned skeletal volume and aggregates them with pixel-features via attention mechanisms.

### [Residual-NeRF](18113-residual-nerf.md)
Residual-NeRF is a method for improving depth perception and training speed for transparent objects in mostly static scenes by leveraging a static scene prior. It solves the ambiguity of depth estimation for transparent objects, where standard NeRFs struggle due to multiple surfaces radiating light along a ray, leading to noisy depth maps and perceived holes that hinder robotic grasp planning. Unlike methods that require pre-defined spatial areas for fusion (e.g., Block-NeRF, NeRFusion), Residual-NeRF learns to mix a pre-trained background NeRF and a residual NeRF at every point in space using a learned gating network.

### [MZEN (Multi-zoom Enhanced NeRF)](18116-mzen-multi-zoom-enhanced-nerf.md)
MZEN is a framework for 3-D reconstruction from images with unknown camera poses that explicitly models optical zoom as a learnable intrinsic parameter and uses cross-zoom pose registration to align non-overlapping zoomed-in and zoomed-out views. It solves the problem of stable training across multiple zoom levels without requiring high-resolution images or external pose initialization. Unlike [CamP](15539-ue4-nerf.md) which perturbs focal length per view, or [Mip-NeRF](15532-rip-nerf.md) and [Zip-NeRF](15532-rip-nerf.md) which rely on image pyramids, MZEN keeps every frame at native resolution and optimizes zoom parameters jointly with the NeRF weights.

### [UV Volumes](18151-uv-volumes.md)
UV Volumes is a method for real-time, editable free-view rendering of human performers that decouples smooth 3D geometry and density from high-frequency appearance. It differs from standard NeRF by encoding smooth density and UV coordinates in a 3D feature volume while storing detailed appearance in a 2D Neural Texture Stack (NTS), enabling real-time performance and retexturing.

### [Switch-NeRF](18158-switch-nerf.md)
Switch-NeRF++ is a large-scale Neural Radiance Field method that uses a trainable decomposition and heterogeneous representation to scale NeRF to large scenes, extending Switch-NeRF with hash-based networks and heterogeneous experts. It employs a Heterogeneous Mixture of Hash Experts (HMoHE) framework where both the gating network and the experts are hash-based, allowing the network to select a suitable resolution range for each scene part. Unlike Switch-NeRF which uses homogeneous experts and an MLP-based gating network, Switch-NeRF++ uses heterogeneous hash experts and a hash-based gating network to dynamically route queries to appropriate resolution grids.

### [Low-Light NeRF (LLNeRF)](18162-low-light-nerf-llnerf.md)
Low-Light NeRF (LLNeRF) is an unsupervised method that enhances scene representation and synthesizes normal-light novel views directly from sRGB low-light images by decomposing radiance field learning into view-dependent and view-independent components. It solves the problem of training NeRF on camera-finished low dynamic range (LDR) sRGB images, which suffer from low signal-to-noise ratio, non-linearity introduced by the camera ISP, and color distortion. Unlike HDR-NeRF, which requires RAW data with exposure metadata, LLNeRF operates on standard sRGB images without ground truth supervision, modeling the ISP pipeline within the network.

### [Compression and Model Efficiency](18217-compression-and-model-efficiency.md)
This family covers methods that reduce the memory footprint and computational cost of NeRF representations through low-rank tensor approximations, weight decomposition, and pruning [CCNeRF](15008-ccnerf.md), [Low Rank NeRF Approximation with ADMM](18163-low-rank-approximation-with-admm.md),. It differs from sibling methods that modify architecture for specific tasks by focusing strictly on compressing the existing radiance field parameters or optimizing them under rank constraints [CCNeRF](15008-ccnerf.md), [Low Rank NeRF Approximation with ADMM](18163-low-rank-approximation-with-admm.md). [G] This reduces the number of trainable parameters from $O(N^2)$ or $O(N^3)$ to $O(N \cdot r)$ or $O(N \cdot r^2)$ where $r$ is the rank.

### [Dynamic and Articulated Scene Modeling](18219-dynamic-and-articulated-scene-modeling.md)
Dynamic and Articulated Scene Modeling extends static radiance fields to handle time-varying geometry and non-rigid deformations, primarily in human performance capture [G]. It solves the ill-posed problem of mapping dynamic 3D points to a static representation by introducing canonical spaces, skeletal constraints, or surface-relative encodings [G]. Unlike static methods, these approaches must jointly optimize or infer pose, deformation, and appearance from monocular or sparse multi-view video inputs [G].

### [View-Dependent Appearance and Lighting](18221-view-dependent-appearance-and-lighting.md)
This family of methods extends the standard NeRF radiance field representation by explicitly modeling how scene appearance varies with the viewing direction, enabling effects such as specular reflections, transparency, and view-dependent lighting. While standard NeRFs often bake shading and shadows into the observed color, making them non-relightable and prone to capturing spurious correlations, these methods separate intrinsic scene properties (albedo, geometry) from extrinsic lighting and view-dependent effects. This separation allows for relighting, novel view synthesis under different lighting conditions, and physically-based material estimation.

### [Generative and GAN-Based Methods](18360-generative-and-gan-based-methods.md)
Generative and GAN-Based Methods synthesize novel 3D objects or scenes by optimizing a radiance field to match the distribution of a pretrained 2D Generative Adversarial Network (GAN) or by directly training a generative radiance field with adversarial losses. Unlike standard NeRF which reconstructs a specific static scene from images, these methods generate new instances by sampling latent codes, enabling data augmentation and novel view synthesis of unseen objects.

### [Training Configuration and Optimization](18409-training-configuration-and-optimization.md)
This article defines the hyperparameter configurations, optimization schedules, and loss function variants required to train Neural Radiance Field (NeRF) models effectively. It addresses the problem of converging implicit neural representations from sparse or unconstrained 2D image collections by specifying learning rates, sampling strategies, and architectural regularizations. What sets this family apart from the underlying [Model Architecture and Encoding](13550-model-architecture-and-encoding.md) is the focus on the *training dynamics*—such as learning rate schedules, loss weighting, and sparsity constraints—rather than the network topology itself.

### [Surface and SDF-Based Representations](18411-surface-and-sdf-based-representations.md)
This family replaces the standard volume density $\sigma$ output of canonical NeRF with a Signed Distance Function (SDF) or occupancy field to represent geometry. By modeling the scene as an implicit surface, these methods achieve sharper geometry extraction and better surface definition than density-based approaches, which often suffer from "fuzzy" boundaries.

### [MLP Network Architecture](18457-mlp-network-architecture.md)
This family defines the Multi-Layer Perceptron (MLP) architectures used to represent neural radiance fields, mapping encoded 3D spatial coordinates and viewing directions to volume density and RGB color. It solves the problem of learning continuous, view-dependent appearance and geometry from discrete 2D images. What sets this family apart from its siblings is the specific configuration of fully-connected layers, skip connections, and activation functions used to process the high-dimensional positional encodings generated by the parent [Model Architecture and Encoding](13550-model-architecture-and-encoding.md).

### [Position Encoding and Embedding](18459-position-encoding-and-embedding.md)
Position encoding and embedding methods transform low-dimensional input coordinates (position $\mathbf{x}$ and view direction $\mathbf{d}$) into high-dimensional feature spaces to enable neural networks to approximate high-frequency functions, thereby overcoming the spectral bias inherent in standard MLPs. This family distinguishes itself from generic coordinate mapping by explicitly designing the frequency spectrum, spatial reference frames, or ray-space parameterizations to address specific challenges such as few-shot overfitting, sparse view interpolation, and articulated motion.

### [NeuRay](18646-neuray.md)
NeuRay is an occlusion-aware image-based rendering method that represents a scene by estimating occlusion probabilities (hitting probabilities) from input views to arbitrary 3D points, enabling efficient novel view synthesis. Unlike standard radiance field methods that treat volume density as a continuous scalar, NeuRay explicitly models visibility to handle feature inconsistency caused by occlusions, particularly in sparse or 360-degree captures. It differs from sibling methods like IBRNet by integrating a visibility encoder and using spherical harmonics to fit anisotropic colors, allowing for robust rendering from unstructured inputs.

### [AReT (Anatomy-Regularized TensoRF)](18648-aret-anatomy-regularized-tensorf.md)
AReT is a scene-specific (patient-specific) inverse solver that reconstructs 3D lung nodule volumes from only three orthogonal Digitally Reconstructed Radiographs (DRRs: coronal, sagittal, axial) using a Tensorial Radiance Field (TensoRF) representation. It solves the problem of sparse-view medical reconstruction where standard TensoRF fails due to a density-parameterization dead zone that suppresses gradients. Unlike generative-prior approaches like MedNeRF, AReT is prior-free during training, relying instead on a corrected density initialization and anatomy-regularized loss to ensure physically plausible attenuation coefficients.

### [AutoRF](18649-autorf.md)
AutoRF is a method for learning 3D object radiance fields from a single view observation without prior geometry knowledge (e.g., CAD models). It disentangles object representation into shape and appearance components using a normalized, object-centric encoding conditioned on 3D bounding boxes and 2D masks. Unlike multi-view methods like PixelNeRF, AutoRF operates on a single image, using an encoder to generate shape and appearance codes and a decoder to render views via volumetric rendering, enabling zero-shot generalization to new objects.

### [HALO](18659-halo.md)
HALO is a few-shot view synthesis method that prevents overfitting in neural radiance fields by regularizing a high-frequency NeRF (Hi-NeRF) using a low-frequency NeRF (Lo-NeRF). It leverages the distinct interpolation and extrapolation properties of low- and high-frequency representations to enforce geometric consistency without requiring extra semantic supervision. Unlike sibling methods such as [DietNeRF](18409-training-configuration-and-optimization.md) which rely on CLIP-based semantic priors, HALO maintains better lighting consistency and operates purely on photometric data, making it suitable for sparse input scenarios.

### [Neural LiDAR Fields](18679-neural-lidar-fields.md)
Neural LiDAR Fields (LiDAR4D) reconstructs LiDAR point clouds into an implicit continuous representation by modeling geometric depth, reflection intensity, and ray-drop probability, differing from standard NeRFs which optimize for photometric RGB loss. It distinguishes itself by combining coarse-resolution multi-planar features with high-resolution hash grid representations to handle large-scale dynamic driving scenarios efficiently, allowing for joint optimization of visual and LiDAR data streams.

### [NeRF Optimization via SDS](20275-nerf-optimization-via-sds.md)
NeRF Optimization via SDS (Score Distillation Sampling) optimizes the parameters of a Neural Radiance Field (NeRF) to match a user-provided text prompt by leveraging gradients from a pretrained 2D text-to-image diffusion model. Unlike standard NeRF training which minimizes pixel-wise reconstruction error against ground-truth images, this method treats the NeRF as a differentiable renderer within an optimization loop, using the diffusion model as a perceptual prior to guide the generation of novel 3D geometry and appearance. It distinguishes itself from sibling methods by enabling text-driven 3D generation without 3D supervision.

### [NeX-MPI](20277-nex-mpi.md)
NeX-MPI is a real-time view synthesis method that combines Multiplane Image (MPI) parameterization with a neural scene representation to model view-dependent effects. It solves the limitation of standard MPIs, which assume brightness constancy and fail on non-Lambertian surfaces (e.g., glass, ceramics) by appearing dull or breaking down. Unlike NeRF, which renders slowly via volumetric integration, NeX-MPI caches a 3D MPI grid optimized via MLPs, enabling real-time rendering in fragment shaders while preserving view-dependent appearance.

### [AS-NeRF](20280-as-nerf.md)
AS-NeRF is a 3D reconstruction method designed to improve accuracy and quality from single or few views by integrating multi-task learning feature extraction, adaptive scale adjustment with spectral sensing, and a conditional diffusion model. It addresses the limitations of existing methods that either focus solely on feature extraction (e.g., Transformer-based NeRF) or sparse viewpoint fusion (e.g., SparseFusion) by explicitly modeling local details and geometric consistency. Unlike [SparseFusion](14964-suds.md) which combines view-conditional diffusion and distillation, or [PixelNeRF](14964-suds.md) which averages pixel features, AS-NeRF uses an adaptive scale function to modulate sample contributions based on scene complexity.

### [CoNeRF](20282-conerf.md)
CoNeRF is a learning framework that extends Neural Radiance Fields to allow intuitive and interpretable user control over specific scene attributes (e.g., facial expressions) via few-shot supervision, treating attributes as latent variables regressed by the network. Unlike standard NeRFs that reconstruct a single static scene, or HyperNeRF which learns an arbitrary attribute space that cannot be supervised or related to specific local attributes, CoNeRF provides direct, interpretable control over localized appearance using sparse 2D mask annotations. It is domain-agnostic, avoiding task-specific architectures, and enables editing of attributes like pose or expression without retraining the entire field.

### [NARFD with Autoencoder for Appearance Variation](20290-narfd-with-autoencoder-for-appearance-variation.md)
This method extends the Neural Articulated Radiance Field (NARFD) framework by integrating a 2D CNN-based autoencoder to model shape and appearance variations across multiple articulated objects (specifically human bodies). It solves the problem of single-view reconstruction and appearance transfer by learning a disentangled latent representation from a single input RGB image, allowing for the independent control of camera viewpoint, bone parameters, pose, and color appearance. Unlike standard NeRF variants that optimize per-scene, this approach learns a global generative model capable of synthesizing novel instances.

### [MTL-FeatureNet](20296-mtl-featurenet.md)
MTL-FeatureNet is a feature extraction network designed for multi-task learning that improves the accuracy and relevance of features extracted from a single 2D input view. It addresses the challenge of adapting a unified feature representation to diverse task requirements by integrating task dynamic attention modulation and a dynamic adapter layer. Unlike standard NeRF encodings which map coordinates to radiance properties directly, MTL-FeatureNet operates on 2D image features to support multiple downstream tasks simultaneously, distinguishing it from sibling architectures like [SUDS](14964-suds.md) which use separate branches for different modalities.

### [Adaptive-Scale and Spectral-Aware NeRF](20309-adaptive-scale-and-spectral-aware-nerf.md)
Adaptive-Scale and Spectral-Aware NeRF is a neural radiance field variant that integrates an adaptive scale function to modulate sample point contributions and a spectral decoder to handle multispectral information, including infrared and ultraviolet wavelengths, for improved color fidelity. It builds on [Model Architecture and Encoding](13550-model-architecture-and-encoding.md) by replacing the standard color prediction with a wavelength-aware mechanism and adjusting density integration weights via local feature analysis.

### [Panoptic and Semantic Scene Representations](20495-panoptic-and-semantic-scene-representations.md)
Panoptic and Semantic Scene Representations extend standard radiance fields by predicting semantic class distributions and instance identifiers alongside density and color, enabling high-level scene understanding and object-centric manipulation. Unlike standard NeRFs that only model geometry and appearance, these methods decompose the scene into "things" (discrete objects) and "stuff" (amorphous background) or predict part-level semantics, allowing for tasks such as object deletion, duplication, and category-level generalization.

### [Satellite and Remote Sensing NeRFs](20496-satellite-and-remote-sensing-nerfs.md)
Not given in the sources.

### [NeO 360](20971-neo-360.md)
NeO 360 is a generalizable neural field method for few-shot novel view synthesis of unbounded 360-degree outdoor scenes. It operates in a zero-shot manner, taking one or a few input images to render novel viewpoints without optimizing a per-scene latent code or using inference-time GAN inversion. It distinguishes itself from sibling methods like [PixelNeRF](14964-suds.md) by combining a global image-conditional tri-planar representation with local residual features and separate foreground/background rendering MLPs, enabling strong scene priors for sparse view inputs.

### [SCADE](20972-scade.md)
SCADE is a NeRF reconstruction approach that regularizes optimization with multimodal depth estimates from an ambiguity-aware prior, resolving ambiguities via a space carving loss that fuses estimates across views to achieve globally consistent 3D reconstruction. Unlike Dense Depth Prior (DDP) which uses a unimodal Gaussian prior and 2D supervision on moments, SCADE uses a multimodal prior and 3D supervision on the full distribution. This allows it to clear "dust" in space by supervising the full distribution of ray termination distances rather than just moments.

### [PBR-NeRF](21721-pbr-nerf.md)
PBR-NeRF is an inverse rendering model that jointly estimates scene geometry, materials, and illumination by extending the NeILF/NeILF++ neural fields framework with two novel physics-based priors: Conservation of Energy and NDF-weighted Specular Losses. It solves the material-lighting ambiguity inherent in standard NeRF approaches by enforcing physical validity on the estimated BRDF, preventing issues such as specular highlights being "baked in" to the diffuse albedo. Unlike NeILF/NeILF++, which use a Lambertian prior that assumes perfectly rough, non-metallic materials and suppresses specularities, PBR-NeRF explicitly models metallicness and roughness, enabling realistic rendering of reflective surfaces.

### [Static Semantic Human NeRF](21733-static-semantic-human-nerf.md)
Static Semantic Human NeRF (H-NeRF) is a neural rendering method that co-learns a volumetric radiance field for free-viewpoint rendering and a signed distance function (SDF) for 3D geometric reconstruction of a static human subject. It solves the problem of poor generalization and ambiguous opacity fields in standard NeRF when trained on sparse multi-view images of humans, by leveraging an implicit parametric human model (imGHUM) as a geometric prior. Unlike standard NeRF, which relies solely on photometric consistency, H-NeRF couples the radiance field with a parametric model to enforce anatomical plausibility and semantic consistency.

### [Attention-Based Volumetric Rendering](21739-attention-based-volumetric-rendering.md)
Attention-Based Volumetric Rendering (ABLE-NeRF) replaces the physics-based volumetric rendering (VR) equation with a self-attention mechanism to handle view-dependent effects, glossy surfaces, and translucent objects. Unlike standard NeRF which assigns weights based on density $\sigma$ and interval $\delta$, ABLE-NeRF uses Transformer attention to determine point importance, and unlike Ref-NeRF which relies on normal prediction, it avoids the ill-posed nature of predicting normals in an object's interior. It sets itself apart from other transformer-based methods by using hierarchical volume embeddings and learnable embeddings to capture global scene context and view-dependent appearance simultaneously.

### [Deferred NeRF](21757-deferred-nerf.md)
Deferred NeRF is a radiance field optimization method that decouples view-independent geometry and diffuse color from view-dependent specular effects to enable real-time rendering via voxel grid baking. Unlike standard NeRF which predicts final color directly from position and direction, Deferred NeRF separates the rendering equation into an accumulated diffuse component and a residual specular component predicted by a secondary network, allowing the diffuse properties to be stored in a dense 3D grid for fast lookup.

### [Neural Surface Reconstruction and Rendering for LiDAR-Visual Systems](21804-neural-surface-reconstruction-and-rendering-for-lidar-visual.md)
M2Mapping is a method that integrates LiDAR-derived occupancy maps with neural radiance fields (NeRF) and signed distance fields (SDF) to reconstruct geometry and render appearance for LiDAR-visual systems. It distinguishes itself from standard NeRF approaches by using visible-aware occupancy classification to handle unknown spaces and structure-aware sampling to adaptively capture sharp geometric details, particularly for slim objects.

### [CRAYM](22860-craym.md)
CRAYM is a method for 3D neural field optimization that jointly optimizes camera poses and an implicit field to handle noisy camera poses. It constructs a 3D feature volume $V$ and optimizes it by matching rays across different images, using key rays for structural constraints and auxiliary rays for local context. Unlike standard NeRF approaches that assume known poses, CRAYM parameterizes transformation matrices as variables in the joint optimization, allowing it to recover geometry and appearance even when initial pose estimates are inaccurate.

### [ColNeRF](23138-colnerf.md)
ColNeRF is a generalizable sparse-input neural radiance field method that achieves accurate 3D modeling with color and geometric consistency without auxiliary supervision. It sets itself apart from siblings like [PixelNeRF](16433-multi-view-aggregation-with-skeletal-features.md) by replacing direct pixel feature averaging with Collaborative Cross-View Volume Integration (CCVI) to implicitly correct occluded regions, and from methods like [DS-NeRF](20280-as-nerf.md) by using self-supervised ray regularization instead of inaccurate depth labels.

### [VolSDF Model Architecture](23580-volsdf-model-architecture.md)
VolSDF is a volume rendering framework for neural implicit surfaces that represents volume density as a transformed version of the signed distance function (SDF) to the learned surface geometry, allowing disentanglement of geometry (density) and radiance field. Unlike IDR, which requires object masks and sets pixel color based on a single intersection point leading to extraneous surface parts, VolSDF does not require masks and uses the volume rendering integral to avoid this issue. It builds on [Model Architecture and Encoding](13550-model-architecture-and-encoding.md) by replacing the density output with an SDF and applying a differentiable transformation to derive opacity.

### [Hierarchical and Sparse Grid Representations](23610-hierarchical-and-sparse-grid-representations.md)
Hierarchical and Sparse Grid Representations discretize 3D space into learnable structures (grids, octrees, or hash tables) to store density and feature vectors, enabling efficient differentiable querying via interpolation. Unlike coordinate-based MLPs that map coordinates to color directly, this family decouples spatial storage from decoding, allowing for faster rendering and explicit scene manipulation. It distinguishes itself from sibling planar representations (e.g., Triplanes) by maintaining full 3D spatial coherence and from pure neural implicit methods by offering explicit control over spatial resolution.

### [LiveScene](23710-livescene.md)
LiveScene is a scene-level language-embedded interactive radiance field that reconstructs and controls multiple articulated objects in complex physical scenes from monocular video. It solves the challenge of high-dimensional interaction space modeling by decomposing the scene into local deformable fields, allowing independent control of individual objects and handling topology changes induced by interactions. Unlike joint modeling approaches like CoNeRF or static language fields like LERF, LiveScene projects high-dimensional $(3+\alpha)$-dimensional interaction variables into local 4D fields, enabling real-time manipulation of dynamic scenes.

### [NeuTex](23768-neutex.md)
NeuTex is a neural scene representation that explicitly disentangles scene geometry from appearance by representing geometry as a continuous 3D volume and appearance as a continuous 2D texture map in an automatically discovered UV space. Unlike standard NeRF, which uses a single MLP to bake both density and radiance into a 3D volume, NeuTex uses three separate MLPs for geometry, texture mapping, and texture, enabling disentangled modeling and texture editing.

### NeRF-Liif
A baseline method that super-resolves LR images using a pretrained LIIF (Local Implicit Image Function) model, which requires LR-HR pairs for training and introduces external data priors [S2]. On synthetic Blender dataset, LIIF may produce false predictions due to domain mismatch with its external training set [S2]. On real-world LLFF dataset, LIIF tends to be oversmooth and may predict false color or geometry [S2]. Performance metrics: Blender ×2 (100x100): PSNR 27.07, SSIM 0.919, LPIPS 0.067; Blender ×4 (100x100): PSNR 25.36, SSIM 0.885, LPIPS 0.125; Blender ×2 (200x200): PSNR 28.81, SSIM 0.934, LPIPS 0.058; Blender ×4 (200x200): PSNR 27.34, SSIM 0.912, LPIPS 0.096; LLFF ×2: PSNR 26.81, SSIM 0.823, LPIPS 0.145; LLFF ×4: PSNR 24.76, SSIM 0.723, LPIPS 0.292 [S2].

### SGC Optimization Framework
The loss function is defined as $L = L_{rgb1} + \gamma_{sh} L_{rgb2} + \gamma_{sm} L_{normalSM} + \gamma_{bi} L_{bilateral}$, where $L_{rgb1}$ is standard rendering loss, $L_{rgb2}$ is physical shading loss [S3]. The optimizer is Adam with an initial learning rate of $5 \times 10^{-4}$, batch size of 1024 sampled rays, and 128 sampled 3D points per ray [S3]. Unlike standard NeRF which may blur boundaries, this method addresses limitations in elevation and structural recovery by ensuring smoothness in flat areas and sharpness at building edges [S3].

### ViiNeuS Hybrid Architecture
The MLP size for both $F^h_\Theta$ and $F^c_\Theta$ uses 2 layers with 64 hidden units [S4]. Positional encoding uses hash encoding with default parameters from InstantNGP, while directional encoding uses spherical harmonics [S4]. The first MLP ($F^h_\Theta$) takes positional embedding $emb_x(x)$ as input and outputs a tuple $(\sigma, f(x), h)$, where $\sigma$ is density, $f(x)$ is SDF value, and $h$ is a latent vector [S4]. The second MLP ($F^c_\Theta$) takes directional embedding $emb_d(d)$, latent vector $h$ from $F^h_\Theta$, and normal vector $-\vec{n}$ derived from SDF gradient as input, and outputs color $c$ [S4].

### Generalized PlenOctree Fusion
A method to boost NeRF rendering by modifying the MLP outputs to Spherical Harmonic (SH) coefficients $z$ and density $\sigma$, cached in PlenOctree leaves. It uses a generalized neural rendering network $\Psi$ to infer implicit volumes without per-scene training [S5]. A generalized neural rendering network $\Psi$ takes images of adjacent views of a target view as inputs and infers an implicit volume with regard to the target view [S5]. Directly query colors and densities of sample points corresponding to leaves in PlenOctree from this volume without per-scene training [S5]. The colors and densities predicted by $\Psi$ are specific to the target view direction. To obtain a view-independent PlenOctree, multiple target views must be sampled and fused [S5].

### Neural Field Classifier (NFC)
A framework that formulates neural field methods as classification tasks rather than regression tasks by encoding continuous regression targets into high-dimensional discrete encodings and optimizing a classification loss [S6, unofficial]. Encode the continuous color target value $y$ into an 8-bit binary vector using BinaryEnc(y) [S6, unofficial]. Modify the final layer of the MLP to output 24 logits (3 channels × 8 bits) representing predicted probabilities $\hat{y}$ for each bit [S6, unofficial]. The weight for the classification loss term $\lambda$ is in the range [0.1, 100], and the constant for numerical stability $\epsilon$ is 0.001 [S6, unofficial]. Predicted probability $\hat{C}$ may exceed 1 due to volume rendering, causing gradient explosion in BCE loss. This is handled by clamping $\hat{C}$ to $1 - \epsilon$ [S6, unofficial]. If ground truth $C$ is not in (0, 1), the gradient of the clamped BCE loss vanishes. This is handled by adding an MSE loss term to ensure updates [S6, unofficial]. NFC uses classification loss (BCE) while NFR uses regression loss (MSE). NFC outperforms NFR with nearly free extra computational cost [S6, unofficial]. NFC uses binary encoding (8 bits per channel) instead of naive one-hot encoding (256 classes per channel) to reduce memory/computation and preserve ordinal information [S6, unofficial].

### RT-NeRF
A neural radiance field variant designed to effectively model a rigidly transformed object part [S7]. Converts a global 3D location into a local coordinate system using an explicit transformation [S7]. Uses the local 3D location to estimate density (implementation details referenced in main paper Eqs. 9 and 10) [S7]. Takes the 6D vector $se(3)$ representation $\xi$ of transformation $l$ as a network input to estimate RGB color $c$ [S7]. Differs from RP-NeRF by using explicit local coordinate transformation for density estimation instead of feeding global 3D location and $\xi$ directly to the network [S7]. Differs from RT-NeRF w/o $\xi$ by including the transformation $\xi$ as a network input for color estimation [S7]. Evaluated on a synthetic dataset of a rigidly transforming bulldozer rendered in Blender [S7].

### Hierarchical Volume Sampling with Coarse-Fine Feature Propagation
A strategy to retain ray representation from the coarse network by reusing the coarse ray token as the input fine ray token, avoiding the quadratic cost of scaling up to $N_c + N_f$ samples in every transformer layer of the fine network [S8]. Use $N_c$ stratified samples for the coarse network [S8]. Generate $N_f = N_c$ samples for the fine network by sampling from the attention weights of the output coarse ray token at state $R^C_L$ (after $L$ layers attending to all coarse volume embeddings) [S8]. Unlike mip-NeRF which discards coarse sample information entirely and uses only $N_f$ samples for final computation, ABLE-NeRF retains coarse information to inform the fine network [S8]. Typical values: $N_c = 96$, $N_f = 96$ [S8].

### Nerflet Representation
A scene representation using $N$ local 'nerflets', where each nerflet $i$ has: 1) 9 pose parameters defining an influence function $g_i$, 2) a tiny MLP $f_i$ generating density and radiance, 3) a single semantic logit vector $s_i$, and 4) an instance ID $In_{si}$ [S9]. Influence function $g_i(x) = \eta \exp(-\frac{1}{2\tau} (x - \mu_i)^T \Sigma_i^{-1} (x - \mu_i))$, where $\mu_i$ is the center, $\Sigma_i$ is a 6-DOF covariance matrix determined by 3 Euler angles and 3 radii (reciprocal of variance), $\eta=5$ is a scaling hyper-parameter, and $\tau$ is a scheduled temperature hyper-parameter reduced after each epoch [S9]. Pose parameters: 9 values per nerflet consisting of a 3D center $\mu_i$, 3 axis-aligned radii, and 3 rotation angles [S9]. $\eta$: Scaling hyper-parameter for the influence function, set to 5 for all experiments [S9]. $\tau$: Scheduled temperature hyper-parameter controlling falloff hardness, reduced after each training epoch to minimize overlap [S9]. Uses a single semantic logit vector $s_i$ per nerflet instead of an MLP to encode semantics, reducing MLP capacity burden and providing inductive bias for 3D spatial consistency [S9].

### Adaptive Scale NeRF
An improvement on standard NeRF designed to increase accuracy and efficiency of 3D reconstruction by dynamically adjusting rendering accuracy according to scene complexity and detail requirements [S10]. Combine with adaptive scale function to adaptively adjust the contribution of each sample point in the NeRF rendering process [S10]. Adaptive scaling function: $S(FA(G,x(t)),G) = \text{sigmoid}(\beta \cdot D_{complex}(CT(FA(G,x(t)),G)))$, where $x(t)$ is 3D location, $FA$ is estimation function for local features, $G$ is global feature, $CT$ is co-feature merging function, $D_{complex}$ is deep neural network for complexity analysis, and $\beta$ is learnable scaling factor [S10]. $\beta$: A learnable scaling factor that regulates the sensitivity of the scale-adaptive function, balancing detail preservation and computational efficiency [S10]. $D_{complex}$: A deep neural network used to analyze integrated local and global features, whose output determines the degree of scale tuning [S10].

### Pixel-Aligned Avatars
Leverages local conditioning in a volume rendering framework where features are aggregated over multiple views, and a MLP produces color and density fields [S11, unofficial]. Leverage local conditioning in a volume rendering framework [S11, unofficial]. Aggregate features over multiple views [S11, unofficial]. Use a MLP to produce color and density fields that are rendered as in NeRF [S11, unofficial].

### NeRF Baseline with DINO
Extend NeRF with an extra head to predict view-independent DINO feature in parallel with density [S12].

### MI-NeRF Multiplicative Interaction Module
A module that learns multiplicative interactions between facial expression vectors and identity codes to disentangle identity and expression information [S13, unofficial]. $M(\mathbf{e}, \mathbf{i}) = \mathbf{C} [(\mathbf{U}_1 \mathbf{e}) * (\mathbf{U}_2 \mathbf{i})] + \mathbf{W}_2 \mathbf{e} + \mathbf{W}_3 \mathbf{i}$, where $*$ is the Hadamard (element-wise) product, $\mathbf{e} \in \mathbb{R}^d$ is the expression vector, $\mathbf{i} \in \mathbb{R}^d$ is the identity vector, $\mathbf{U}_1 \in \mathbb{R}^{k \times d}$, $\mathbf{U}_2 \in \mathbb{R}^{k \times d}$, $\mathbf{C} \in \mathbb{R}^{d \times k}$, $\mathbf{W}_2 \in \mathbb{R}^{d \times d}$, $\mathbf{W}_3 \in \mathbb{R}^{d \times d}$ are learnable parameters, $d=79$, and $k=8$ [S13, unofficial]. $k$: The rank of the low-rank matrices U1 and U2. Typical value: 8 [S13, unofficial]. $d$: The dimension of the expression and identity vectors. Typical value: 79 [S13, unofficial]. Module M uses Hadamard product of projected vectors, whereas variant A7 uses a 3D tensor $W \in \mathbb{R}^{d \times d \times d}$ requiring $d^3$ parameters compared to M's $d^2$ [S13, unofficial].

### Orientation Loss Regularizer
A regularizer to encourage normal vectors of the density field to face toward the camera when visible, preventing geometry from appearing to face backwards [S14, unofficial]. $L_{orient} = \sum_i \text{stop grad}(w_i) \max(0, n_i \cdot v)^2$ [S14, unofficial]. $v$ is the direction of the ray (viewing direction) [S14, unofficial]. A stop-gradient is placed on the rendering weights $w_i$ to prevent unintended local minima where the generated object shrinks or disappears [S14, unofficial].

### Feed-forward NeRF
An architecture presented by Yu et al. that conditions NeRF on image inputs using a fully convolutional method to learn a scene prior [S15]. Performs feed-forward view synthesis from a small number of viewpoints, even as few as one [S15].

### Tera-NeRF
A radiance field representation introduced by Kulhanek and Sattler that adjusts to 3D geometry priors given as a sparse point cloud [S15].

### Light Field Networks
Enhances rendering speeds by training the MLP itself [S16, unofficial].

## Background
The canonical NeRF architecture was introduced by Mildenhall et al. in 2020 [S1]. It established the standard approach of using a fully-connected MLP with positional encoding to represent continuous 3D scenes. The specific configuration of 63 position features and 27 direction features, along with the two-branch network structure, became the de facto baseline for subsequent research in neural radiance fields [S1]. The method has been widely adopted in computer vision and graphics for novel view synthesis, 3D reconstruction, and increasingly, for generative 3D tasks [S1].

## Sources
[S1] MZEN: Multi-zoom Enhanced NeRF for 3-D Reconstruction with Unknown Camera Poses, https://link.springer.com/article/10.1007/s10994-026-07114-x, fetched 2026-10-26
[S2] NeRF-SR: High-Quality Neural Radiance Fields using Supersampling, https://arxiv.org/abs/2112.01759v3, fetched 2026-10-26
[S3] Shading and Geometric Constraint Neural Radiance Field for DSM Reconstruction from Multi-View Satellite Images, https://doi.org/10.3390/rs18071091, fetched 2026-11-05
[S4] ViiNeuS: Volumetric Initialization for Implicit Neural Surface reconstruction of urban scenes with limited image overlap, https://hal.science/hal-05104965v1, fetched 2026-10-26
[S5] Fourier PlenOctrees for Dynamic Radiance Field Rendering in Real-time, https://doi.org/10.1109/cvpr52688.2022.01316, fetched 2026-10-26
[S6] Neural Field Classifiers via Target Encoding and Classification Loss, https://doi.org/10.48550/arxiv.2403.01058, fetched 2026-10-26 (unofficial)
[S7] Neural Articulated Radiance Field, https://doi.org/10.1109/iccv48922.2021.00571, fetched 2026-10-26
[S8] ABLE-NeRF: Attention-Based Rendering with Learnable Embeddings for Neural Radiance Field, https://doi.org/10.1109/cvpr52729.2023.01589, fetched 2026-10-26
[S9] Nerflets: Local Radiance Fields for Efficient Structure-Aware 3D Scene Representation from 2D Supervision, https://doi.org/10.1109/cvpr52729.2023.00800, fetched 2026-10-26
[S10] 3D reconstruction of toys based on adaptive scaled neural radiation field., https://doi.org/10.7717/peerj-cs.3053, fetched 2026-10-26
[S11] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
[S12] AssetField: Assets Mining and Reconfiguration in Ground Feature Plane Representation, https://doi.org/10.1109/iccv51070.2023.00301, fetched 2026-10-29
[S13] MI-NeRF: Learning a Single Face NeRF from Multiple Identities, https://arxiv.org/abs/2403.19920v2, fetched 2026-10-26 (unofficial)
[S14] DreamFusion: Text-to-3D using 2D Diffusion, https://doi.org/10.48550/arxiv.2209.14988, fetched 2026-10-27 (unofficial)
[S15] A Critical Analysis of NeRF-Based 3D Reconstruction, https://doi.org/10.3390/rs15143585, fetched 2026-10-26
[S16] Neural Rendering and Its Hardware Acceleration: A Review, https://doi.org/10.48550/arxiv.2402.00028, fetched 2026-10-26 (unofficial)
