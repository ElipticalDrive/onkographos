# Model Architecture and Encoding

Also called: Compact NeRF Architecture & Ray Casting, Mildenhall et al. (2021) NeRF, NeRF Model Architecture and Encoding, Wang et al. (2021) NeRF

## Overview
This article defines the foundational neural network architecture and positional encoding scheme used in the original Neural Radiance Fields (NeRF) method [S1, S14]. It solves the problem of mapping continuous 3D spatial coordinates and viewing directions to volume density and view-dependent color by using a Multi-Layer Perceptron (MLP) conditioned on high-frequency positional encodings. It sets itself apart from sibling methods like [Mip-NeRF 360 MLP Architecture](14098-mip-nerf-360-mlp-architecture.md) by using standard hierarchical positional encoding rather than integrated positional encoding, and from [Voxel and Grid Representations](18218-voxel-and-grid-representations.md) by relying entirely on coordinate-based MLPs rather than explicit grid lookups.

## Shared principles
The core mechanism involves transforming low-dimensional input coordinates into a high-dimensional feature space using sinusoidal positional encoding, which is then processed by a fully-connected MLP to predict radiance field properties.

**Positional Encoding**
The input coordinates $\mathbf{x} \in \mathbb{R}^3$ (position) and $\mathbf{d} \in \mathbb{R}^3$ (view direction) are mapped to higher dimensions using a hierarchical positional encoding function $\gamma(\cdot)$ [S1, S14]. For a scalar input $p$, the encoding is defined as:
$$ \gamma(p) = (\sin(2^0 \pi p), \cos(2^0 \pi p), \dots, \sin(2^{L-1} \pi p), \cos(2^{L-1} \pi p)) $$
where $L$ is the number of frequency bands [S14]. This function is applied independently to each of the three components of the position vector and the direction vector [S1, S14].

**Network Architecture**
The radiance field is represented by an MLP that takes the encoded position and direction as input. The architecture typically consists of two branches:
1.  **Density Branch:** Processes the encoded position features to predict volume density $\sigma$.
2.  **Color Branch:** Processes the encoded direction features concatenated with intermediate features from the density branch to predict RGB color $\mathbf{c}$ [S1].

**Ray Sampling**
Rendering requires sampling points along camera rays. A ray is parameterized as $r(\lambda) = o_{world} + \lambda d_{world}$, where $o_{world}$ is the world-space origin and $d_{world}$ is the world-space direction [S1]. The direction is computed from camera intrinsics and extrinsics: $d_{cam} = (\frac{i - c_x}{f_x}, \frac{j - c_y}{f_y}, 1)$, $d_{world} = R d_{cam}$, and $o_{world} = t$ [S1]. Position samples $x$ are generated along the ray within a defined near/far range [S1].

## Subtopics

### [Depth-Guided NeRF Training Configuration](11916-depth-guided-nerf-training-configuration.md)
A training configuration that incorporates depth priors to regularize the optimization process. It distinguishes itself by explicitly integrating depth supervision into the loss function to address scale and geometry ambiguity in sparse-view settings.

### [CLIPNeRF Training Configuration](12073-clipnerf-training-configuration.md)
A text-driven NeRF stylization method that enforces stylization trajectories in the CLIP embedding space. It distinguishes itself by freezing density-related parameters and optimizing radiance parameters to match text prompts.

### [Binary Radiance Fields](13622-binary-radiance-fields.md)
A storage-efficient representation that binarizes feature embedding parameters to $\{-1, +1\}$ using a sign function and backpropagating through a straight-through estimator. It distinguishes itself by reducing model size significantly compared to standard dense MLPs.

### [Mip-NeRF 360 MLP Architecture](14098-mip-nerf-360-mlp-architecture.md)
A two-branch MLP that predicts volume density and view-dependent radiance. It distinguishes itself from standard NeRF by using **integrated positional encoding** to analytically integrate the positional encoding over conical frustums, addressing aliasing.

### [SUDS](14964-suds.md)
A 4D radiance field representation for large-scale dynamic urban scenes that factorizes the scene into three separate hash table data structures for static, dynamic, and far-field radiance. It distinguishes itself by handling temporal dynamics and large scales via hash encoding.

### [Rip-NeRF](15532-rip-nerf.md)
A method that renders high-fidelity anti-aliased images by combining cone casting with an anisotropic 3D Gaussian representation. It distinguishes itself by using Ripmap-Encoded Platonic Solids for efficient anisotropic area-sampling.

### [UE4-NeRF](15539-ue4-nerf.md)
A neural rendering system for real-time interactive rendering of large-scale 3D scenes by integrating NeRF with the Unreal Engine 4 rasterization pipeline. It distinguishes itself by targeting real-time performance in game engines rather than offline training.

### [Panoptic Radiance Field](15849-panoptic-radiance-field.md)
Learns a panoptic 3D volumetric representation from posed RGB images and machine-generated 2D panoptic segmentation masks. It differs from siblings by lifting noisy 2D labels into a consistent 3D representation.

### [Frequency-Modulated Position Encoding](15897-frequency-modulated-position-encoding.md)
A positional encoding scheme for point-based radiance field optimization that provides fine-grained basis function support for frequency modulation of spatial coordinates and view directions. It solves the problem of representing high-frequency details in point clouds.

### [Multi-view Aggregation with Skeletal Features](16433-multi-view-aggregation-with-skeletal-features.md)
Improves pose refinement and feature representation in articulated neural radiance fields by incorporating multi-view constraints. It differs from single-view approaches by averaging joint rotation estimates across views.

### [Neural Light Fields with Ray-Space Embedding](17616-neural-light-fields-with-ray-space-embedding.md)
Maps 4D ray parameters directly to integrated radiance, enabling rendering with a single network evaluation per pixel for small-baseline scenes. It distinguishes itself from NeRF by avoiding hundreds of volumetric samples.

### [Neural Scene Graphs](17706-neural-scene-graphs.md)
Decomposes dynamic, multi-object scenes into a learned hierarchical scene graph with decoupled object transformations. Unlike existing neural rendering methods, it explicitly models object relationships and dynamics.

### [Lightning-NeRF Model Configuration](18065-lightning-nerf-model-configuration.md)
A configuration for efficient neural radiance field training and rendering that utilizes hash-based encoding and specific architectural choices. It distinguishes itself by separating geometry and appearance into distinct encoder networks for independent optimization.

### [Vox-Surf](18110-vox-surf.md)
A voxel-based neural implicit surface representation that combines explicit sparse voxel structures with neural implicit surfaces. It differs from traditional voxel-based methods by enabling efficient surface reconstruction and realistic rendering.

### [Residual-NeRF](18113-residual-nerf.md)
Improves depth perception and training speed for transparent objects by leveraging a static scene prior. It solves the ambiguity faced by standard NeRFs when reconstructing transparent objects by decomposing the scene into static and residual components.

### [MZEN (Multi-zoom Enhanced NeRF)](18116-mzen-multi-zoom-enhanced-nerf.md)
A framework for 3-D reconstruction from images with unknown camera poses and varying optical magnifications, explicitly modeling optical zoom as a learnable intrinsic parameter. It solves geometric inconsistency in pose-free NeRF when magnification changes.

### [Ray-Space Embedding Network with Local Affine Transformation](18128-ray-space-embedding-network-with-local-affine-transformation.md)
A neural light field representation that uses a ray-space embedding network to predict local affine transformations for color computation. It differs from standard NeRF by parameterizing view-dependent effects without explicit geometry.

### [SpS-NeRF](18130-sps-nerf.md)
A variant of NeRF adapted for sparse single-date satellite imagery that employs dense depth supervision guided by a cross-correlation similarity metric. It extends Sat-NeRF to enable novel view generation and 3D geometry reconstruction from satellite data.

### [Mip-NeRF Network Architecture](18138-mip-nerf-network-architecture.md)
A compact, fully-connected multilayer perceptron (MLP) designed for neural radiance fields, specifically optimized for efficiency while maintaining structural similarity to the original NeRF architecture. It solves the problem of high computational cost in rendering.

### [Pixel-aligned Volumetric Avatars (PVA)](18139-pixel-aligned-volumetric-avatars-pva.md)
Estimates a volumetric 3D avatar of a human head from a small number of input images. It distinguishes itself from global latent code approaches by using pixel-aligned features for view synthesis of unseen identities and expressions.

### [Neural Deformable Anchor (NeuDA)](18140-neural-deformable-anchor-neuda.md)
A neural implicit representation for high-fidelity surface reconstruction that replaces learnable feature vectors at voxel vertices with optimized 3D positions (anchors). It solves the limitation of standard voxel grids where uniformly distributed features are inefficient.

### [Neural Duplex Radiance Fields (NDRF)](18147-neural-duplex-radiance-fields-ndrf.md)
Distills a pretrained NeRF into a mesh-based representation that enables real-time view synthesis. It solves the high computational cost of standard NeRF rendering by replacing dense volumetric sampling with a two-layer mesh.

### [Skeleton-Relative Encoding](18148-skeleton-relative-encoding.md)
Transforms query locations and view directions into local bone-relative coordinate systems before processing them through a neural radiance field. It differs from standard NeRF by explicitly incorporating domain knowledge of human body part linkage.

### [UV Volumes](18151-uv-volumes.md)
A method for real-time, editable free-view rendering of human performers that decouples smooth volumetric geometry from high-frequency appearance. It encodes density and UV coordinates in a 3D feature volume while storing detailed appearance in a 2D Neural Texture Stack.

### [Low-Light NeRF (LLNeRF)](18162-low-light-nerf-llnerf.md)
An unsupervised method that synthesizes normal-light novel views directly from low-dynamic-range (LDR) sRGB images captured in low-light conditions. It solves the problem of NeRF's inability to handle non-linearity, low signal-to-noise ratio, and color distortion in low-light scenes.

### [Image-Conditional Triplanar Representation](18169-image-conditional-triplanar-representation.md)
Models scenes using three axis-aligned orthogonal planes ($S = [S_{xy}, S_{xz}, S_{yz}]$) to achieve full expressivity at scale. It avoids the cubic memory complexity of voxel-based representations and the dimensionality issues of triplane methods.

### [I2-SDF](18170-i2-sdf.md)
A neural implicit framework for complex indoor scenes that jointly recovers underlying shape, radiance, and material fields from multi-view images. It distinguishes itself by decomposing the radiance field into spatially-varying material and emission components using surface-based, differentiable rendering.

### [SDF-based Volume Rendering with Laplacian CDF](18174-sdf-based-volume-rendering-with-laplacian-cdf.md)
Renders geometry by outputting a Signed Distance Function (SDF) value $s$ and a variance parameter $\beta$ from a neural network, then transforming $s$ into opacity $\sigma$ using a Laplacian Cumulative Distribution Function (CDF). It builds on [Model Architecture and Encoding](1355-model-architecture-and-encoding.md) by replacing the standard density prediction with SDF-based rendering.

### [Compression and Model Efficiency](18217-compression-and-model-efficiency.md)
Addresses the excessive memory footprint and computational cost of standard NeRFs by reducing model size through low-rank approximations, tensor decompositions, and pruning. It differs from sibling methods that modify architecture for specific tasks.

### [Voxel and Grid Representations](18218-voxel-and-grid-representations.md)
Replace coordinate-based Multi-Layer Perceptrons (MLPs) with explicit, differentiable 3D grids (dense or sparse) to store density and appearance features. Unlike MLPs which suffer from slow convergence and limited capacity, grids enable faster optimization and rendering.

### [Dynamic and Articulated Scene Modeling](18219-dynamic-and-articulated-scene-modeling.md)
Extends static radiance fields to handle time-varying geometry and non-rigid deformations, primarily in human performance capture. It solves the ill-posed problem of mapping dynamic 3D points to a static representation by introducing canonical spaces.

### [Lighting and Intrinsic Decomposition](18220-lighting-and-intrinsic-decomposition.md)
Refers to methods that separate a Neural Radiance Field (NeRF) into view-independent intrinsic properties (such as albedo, normals, and density) and view-dependent lighting components. Unlike standard NeRF which outputs raw RGB, these methods enable realistic relighting and scene understanding.

### [View-Dependent Appearance and Lighting](18221-view-dependent-appearance-and-lighting.md)
Extends the standard NeRF representation by explicitly modeling the angular dependence of surface appearance and lighting. This allows the network to predict view-dependent effects such as specular reflections, anisotropic materials, and refraction/reflection in transparent objects.

### [Generative and GAN-Based Methods](18360-generative-and-gan-based-methods.md)
Synthesize novel 3D objects or scenes by optimizing a radiance field to match the distribution of a pretrained 2D Generative Adversarial Network (GAN) or by directly training a generative radiance field with adversarial losses. Unlike standard NeRF which reconstructs specific scenes, these methods generate new instances.

### Triplane NeRF with Diffusion Prior
A framework that uses a triplane NeRF representation where scene codes are optimized jointly with a diffusion prior to handle sparse-view data and ensure end-to-end coherence [S2]. The scene code $\xi_i$ has shape $\mathbb{R}^{3 \times 6 \times 128 \times 128}$ [S2]. The denoising network is a U-Net with 122M parameters [S2]. It uses v-parameterization $\hat{v}_\phi(x(t), t)$ such that $\hat{x} = \alpha(t)x(t) - \sigma(t)\hat{v}$ [S2]. The weighting function uses SNR-based weighting $w(t) = (\alpha(t)/\sigma(t))^2\omega$ [S2].

### EG3D Triplane Representation
Uses a triplane representation $T \in \mathbb{R}^{256 \times 256 \times 96}$, corresponding to 3 axis-aligned planes (xy, xz, yz), each with 32 channels and resolution $256 \times 256$ [S3]. It renders a feature image $I_f \in \mathbb{R}^{32 \times 128 \times 128}$ and raw RGB image $I_{128} \in \mathbb{R}^{3 \times 128 \times 128}$ by accumulating points along rays and performing volume rendering [S3]. It extracts a dense depth map $I_D \in \mathbb{R}^{128 \times 128}$ [S3]. A 2D super-resolution network yields the final output $I_{512} \in \mathbb{R}^{3 \times 512 \times 512}$ via $SuperRes(I_f, I_{128}) = I_{512}$ [S3]. Input images are $512 \times 512$ [S3]. Rendering speed is 42 fps on RTX 3090 [S3].

### NeRF-W Hyperparameter Configuration
Uses a base MLP of 8 layers of 512 hidden units [S4]. Static and Transient/Uncertainty MLPs use 4 layers of 128 hidden units each [S4]. Positional encoding frequencies are 15 for position and 4 for viewing direction [S4]. Sampling points are 512 per ray per model (coarse and fine) during training (1024 total), doubled to 2048 during evaluation [S4]. Appearance embedding dimension $n(a) = 48$ [S4]. Transient object embedding dimension $n(\tau) = 16$ [S4]. L1 regularizer multiplier $\lambda_u = 0.01$ [S4]. Minimum importance threshold $\beta_{min} = 0.03$ [S4]. Optimizer is Adam, batch size 2048, initial learning rate 0.001, decaying ten-fold every 150,000 iterations, over 300,000 iterations total [S4]. Hyperparameters were optimized for the Brandenburg Gate scene of the Phototourism dataset and reused for remaining scenes [S4].

### Explicit Mesh Reconstruction from Neural Fields
An algorithm to extract an explicit 3D mesh representation from the implicit neural radiance field learned by CodeNeRF, allowing for 3D reconstruction from a single input image [S5]. It feeds the center coordinates of a $256^3$ voxel grid to the CodeNeRF MLP to obtain volume density values $\sigma$ [S5]. It applies the Marching Cubes algorithm to the volume density values $\sigma$ to find mesh vertices [S5]. It casts rays along the vertex normals of the generated mesh to associate colors to the vertices [S5].

### NeRF-Liif
A baseline method that super-resolves LR images using a pretrained LIIF (Local Implicit Image Function) model, which requires LR-HR pairs for training and introduces external data priors [S6]. On synthetic Blender dataset, LIIF may produce false predictions due to domain mismatch with its external training set [S6]. On real-world LLFF dataset, LIIF tends to be oversmooth and may predict false color or geometry [S6]. Metrics: Blender $\times 2$ (100x100): PSNR 27.07, SSIM 0.919, LPIPS 0.067; Blender $\times 4$ (100x100): PSNR 25.36, SSIM 0.885, LPIPS 0.125; Blender $\times 2$ (200x200): PSNR 28.81, SSIM 0.934, LPIPS 0.058; Blender $\times 4$ (200x200): PSNR 27.34, SSIM 0.912, LPIPS 0.096; LLFF $\times 2$: PSNR 26.81, SSIM 0.823, LPIPS 0.145; LLFF $\times 4$: PSNR 24.76, SSIM 0.723, LPIPS 0.292 [S6].

### UniSurf
A method unifying volume rendering (radiance fields) and surface rendering (implicit surfaces) to achieve high-quality geometry and sharp textures without mask supervision [S7, unofficial]. It uses an occupancy field where the occupancy at the $i^{th}$ sample corresponds to an indicator of surface presence [S7, unofficial]. It uses volume rendering in the early stage for optimization without mask supervision [S7, unofficial]. It uses surface rendering in the later stage to extract level-set surfaces [S7, unofficial]. Unlike pure surface rendering which requires mask supervision and has blurry texture mapping, this method provides high-quality novel views and sharp textures/colors without mask supervision [S7, unofficial].

### SGC Optimization Framework
Uses a loss $L = L_{rgb1} + \gamma_{sh} L_{rgb2} + \gamma_{sm} L_{normalSM} + \gamma_{bi} L_{bilateral}$, where $L_{rgb1}$ is standard rendering loss, $L_{rgb2}$ is physical shading loss [S8]. Optimizer is Adam. Initial learning rate is $5 \times 10^{-4}$ [S8]. Batch size is 1024 sampled rays [S8]. Samples per ray are 128 sampled 3D points [S8]. Unlike standard NeRF which may blur boundaries, this method addresses limitations in elevation and structural recovery by ensuring smoothness in flat areas and sharpness at building edges [S8].

### Switch-NeRF
A Mixture of Experts (MoE) framework for large-scale scene modeling where both the gating network and the experts are Multi-Layer Perceptrons (MLPs) that learn homogeneous representations [S9]. The gating network G, implemented as an MLP, maps point features into gate values G(x) for a 3D point x [S9]. A Top-1 operation selects the expert E with the highest gate value, and the input point x is sent to that expert E, which is a larger MLP network [S9]. The selected expert E computes a feature fe for point x, which is passed to a small MLP head to predict point density $\sigma$ and point color c [S9]. An appearance embedding (AE) is added to capture view-dependent appearance when predicting the color [S9]. The gating network G is a single MLP network; using more gating networks is avoided to prevent large increases in memory and computation cost [S9]. Switch-NeRF uses an MLP-based gating network, while Switch-NeRF++ uses a hash-based gating network [S9]. Switch-NeRF training time is 44h for 500k steps, compared to Switch-NeRF++*'s 5.5h for 50k steps [S9].

### Uncertain Surface Distillation
A strategy to distill knowledge of previously learned 3D scenes by querying density on a subdivided occupancy grid [S10]. It divides each occupancy grid cell into a 5x5x5 subgrid [S10]. It queries the density of each subgrid point [S10]. It considers a volume cell as a surface for distillation if any subgrid point has density above threshold $\tau$ [S10]. USD occupancy grid resolution is 50x50x50 [S10]. Subgrid division per cell is 5x5x5 [S10]. Density threshold $\tau$ is 3 [S10].

### Occupancy-Aware NeRF
A NeRF variant that jointly learns volumetric density ($\sigma$), RGB color (c), and a binary occupancy value (o) for a 3D point x and viewing direction d, enabling O(1) collision queries via a cached 3D array [S11, unofficial]. At inference, the occupancy grid is cached as a regular 3D array at resolution $\Delta$ to enable O(1) collision queries [S11, unofficial]. The network predicts $f_\theta(x, d) = (\sigma, c, o)$, where $\sigma$ is volumetric density, $c \in \mathbb{R}^3$ is RGB color, and $o \in \{0, 1\}$ is binary occupancy [S11, unofficial]. The loss is $L = L_{rgb} + \lambda_d L_{depth} + \lambda_o BCE(\hat{o}, o^*)$, where $L_{rgb}$ is photometric loss, $L_{depth}$ is depth supervision, $o^*$ is derived from depth observations via volumetric carving, and BCE is binary cross-entropy [S11, unofficial]. The positional encoding is $\gamma(x) = \Pi_{l=1}^{L} h_l(\lfloor x \cdot s_l \rfloor)$, where $h_l$ are learned hash tables at L resolution levels, $s_l$ are per-level scales, and $\Pi$ denotes concatenation [S11, unofficial]. Network architecture: A shared hash-grid backbone (from Instant-NGP [4]) feeds into a separate two-layer MLP head with sigmoid activation for the occupancy branch [S11, unofficial].

### ViiNeuS Hybrid Architecture
MLP Size: Both $F^h_\Theta$ and $F^c_\Theta$ use 2 layers with 64 hidden units [S12]. Positional Encoding: Hash encoding is used for positions, with default parameters from InstantNGP [S12]. Directional Encoding: Spherical harmonics are used to encode viewing directions [S12]. First MLP ($F^h_\Theta$): Input is positional embedding $emb_x(x)$. Output is tuple $(\sigma, f(x), h)$, where $\sigma$ is density, $f(x)$ is SDF value, and $h$ is a latent vector [S12]. Second MLP ($F^c_\Theta$): Inputs are directional embedding $emb_d(d)$, latent vector $h$ from $F^h_\Theta$, and normal vector $-\vec{n}$ derived from SDF gradient. Output is color c [S12].

### Generalized PlenOctree Fusion
A method to boost NeRF rendering by modifying the MLP outputs to Spherical Harmonic (SH) coefficients z and density $\sigma$, cached in PlenOctree leaves. It uses a generalized neural rendering network $\Psi$ to infer implicit volumes without per-scene training [S13]. A generalized neural rendering network $\Psi$ takes images of adjacent views of a target view as inputs and infers an implicit volume with regard to the target view [S13]. It directly queries colors and densities of sample points corresponding to leaves in PlenOctree from this volume without per-scene training [S13]. The colors and densities predicted by $\Psi$ are specific to the target view direction. To obtain a view-independent PlenOctree, multiple target views must be sampled and fused [S13].

### Position Encoding
A mapping technique that transforms low-frequency coordinate representations into high-frequency information using sine and cosine functions, increasing dimensionality to help deep networks approximate high-frequency functions and prevent loss of high-frequency spatial information [S14]. Encoding function: $F_\Theta = F'_\Theta \circ \gamma$, where $\gamma: \mathbb{R} \to \mathbb{R}^{2L}$ is defined as $\gamma(p) = (\sin(2^0 \pi p), \cos(2^0 \pi p), \dots, \sin(2^{L-1} \pi p), \cos(2^{L-1} \pi p))$ [S14]. $\gamma(\cdot)$ is applied separately to the three coordinate values in x and the camera viewing direction d [S14]. Encoding levels L: Set to L=10 for position x and L=4 for direction d [S14].

### View Direction Removal
Removing the view direction condition from color prediction to prevent the model from capturing spurious correlations and dataset bias, thereby improving synthesis consistency [S15, unofficial]. Including view direction gives the model additional freedom to capture spurious correlations and dataset bias, especially with single-view targets [S15, unofficial].

## Background
The original NeRF architecture was introduced by Mildenhall et al. in 2020 [S1]. It established the standard approach of using a fully-connected MLP to map 3D coordinates and viewing directions to density and color, conditioned on hierarchical positional encoding [S1, S14]. This architecture became the baseline for subsequent research in neural radiance fields [S7, unofficial].

## Sources
[S1] NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis, https://arxiv.org/abs/2003.08934, fetched 2026-10-26
[S2] Single-Stage Diffusion NeRF: A Unified Approach to 3D Generation and Reconstruction, https://doi.org/10.1109/iccv51070.2023.00229, fetched 2026-10-29
[S3] Real-Time Radiance Fields for Single-Image Portrait View Synthesis, https://doi.org/10.1145/3592460, fetched 2026-10-27
[S4] NeRF in the Wild: Neural Radiance Fields for Unconstrained Photo Collections, https://doi.org/10.1109/cvpr46437.2021.00713, fetched 2026-10-26
[S5] CodeNeRF: Disentangled Neural Radiance Fields for Object Categories, https://doi.org/10.1109/iccv48922.2021.01271, fetched 2026-10-26
[S6] NeRF-SR: High-Quality Neural Radiance Fields using Supersampling, https://arxiv.org/abs/2112.01759v3, fetched 2026-10-26
[S7] Lecture 7 – Neural Radiance Fields Prof. Dr., https://virtualhumans.mpi-inf.mpg.de/3DVision25/slides25/pdf/Lecture_07_Neural_Radiance_Fields.pdf, fetched 2026-10-27 (unofficial)
[S8] Shading and Geometric Constraint Neural Radiance Field for DSM Reconstruction from Multi-View Satellite Images, https://doi.org/10.3390/rs18071091, fetched 2026-11-05
[S9] Learning Heterogeneous Mixture of Scene Experts for Large-Scale Neural Radiance Fields., https://doi.org/10.1109/tpami.2025.3603305, fetched 2026-10-26
[S10] SCARF: Scalable Continual Learning Framework for Memory‐efficient Multiple Neural Radiance Fields, https://doi.org/10.1111/cgf.15255, fetched 2026-10-26
[S11] NeRF-Nav: Hierarchical Neural Radiance Fields for Real-Time Robot Navigation and Obstacle Avoidance, https://doi.org/10.21203/rs.3.rs-8887612/v1, fetched 2026-10-26 (unofficial)
[S12] ViiNeuS: Volumetric Initialization for Implicit Neural Surface reconstruction of urban scenes with limited image overlap, https://hal.science/hal-05104965v1, fetched 2026-10-26
[S13] Fourier PlenOctrees for Dynamic Radiance Field Rendering in Real-time, https://doi.org/10.1109/cvpr52688.2022.01316, fetched 2026-10-26
[S14] Neural Radiance Fields-Based 3D Reconstruction of Power Transmission Lines Using Progressive Motion Sequence Images., https://doi.org/10.3390/s23239537, fetched 2026-10-26
[S15] StyleNeRF: A Style-based 3D-Aware Generator for High-resolution Image Synthesis, https://doi.org/10.48550/arxiv.2110.08985, fetched 2026-10-26 (unofficial)
