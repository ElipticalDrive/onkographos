# Neural Radiance Fields

Also called: Explicit Radiance Field Representations, Implicit Radiance Field

## Overview
Neural Radiance Fields (NeRF) represent a static 3D scene as a continuous volumetric function that maps a 3D spatial position and a 2D viewing direction to an emitted RGB color and a volume density scalar [S1][S3][S11]. It solves the problem of novel view synthesis by replacing explicit geometric primitives (meshes or point clouds) with an implicit neural representation encoded in the weights of a multi-layer perceptron (MLP) [S1, S5]. Unlike its siblings, NeRF explicitly models view-dependent appearance (specularities, transparency) by conditioning the color output on the viewing direction, while keeping the density view-independent to enforce geometric consistency [S2][S6, unofficial]. It distinguishes itself from nearest alternatives like [Instant Neural Graphics Primitives (Instant NGP)](8637-instant-neural-graphics-primitives-instant-ngp.md) by using a fully implicit global MLP rather than hash-encoded feature grids, resulting in higher quality but significantly slower training and rendering speeds [S1, S5].

## Shared principles
The family relies on the differentiability of the forward rendering operator and the propagation of gradients through the integration steps [44-radiance-field-optimization.md]. The core mechanism is the evaluation of the volume rendering equation and the computation of gradients with respect to scene parameters (density, geometry, transfer functions, camera poses) [44-radiance-field-optimization.md].

The standard emission-absorption model computes the color $\hat{C}(\mathbf{r})$ for a ray $\mathbf{r}$ as [44-radiance-field-optimization.md]:

$$
\hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
$$

where $\sigma_i$ is the density, $\delta_i$ is the step size, $\mathbf{c}_i$ is the color, and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the transmittance [44-radiance-field-optimization.md].

Differentiation strategies fall into three main categories [44-radiance-field-optimization.md]:
1. **Reverse-Mode Automatic Differentiation (AD):** Standard backpropagation through the computational graph. This requires storing $O(kn)$ intermediate states (where $n$ is pixels and $k$ is samples), leading to high memory usage [44-radiance-field-optimization.md].
2. **Adjoint Method:** Computes gradients by propagating adjoint variables backward through the rendering algorithm, evaluating the chain rule from output to input. This has computational complexity largely constant in the number of parameters, making it preferable for optimizing many parameters like the transfer function [44-radiance-field-optimization.md].
3. **Analytic Inversion / Explicit Gradients:** Uses a-priori knowledge of the DVR operations to invert the order of operations or derive closed-form gradients, reducing memory consumption to $O(n)$ by avoiding the storage of intermediate results [44-radiance-field-optimization.md].

## Subtopics

### [NeRF (Neural Radiance Fields)](1185-nerf-neural-radiance-fields.md)
NeRF represents a static 3D scene as a continuous volumetric function that maps a 3D spatial position and a 2D viewing direction to an emitted RGB color and a volume density scalar. It solves the problem of novel view synthesis by replacing explicit geometric primitives (meshes or point clouds) with an implicit neural representation encoded in the weights of a multi-layer perceptron (MLP). Unlike its siblings, NeRF explicitly models view-dependent appearance (specularities, transparency) by conditioning the color output on the viewing direction, while keeping the density view-independent to enforce geometric consistency. It distinguishes itself from nearest alternatives like [Instant Neural Graphics Primitives (Instant NGP)](8637-instant-neural-graphics-primitives-instant-ngp.md) by using a fully implicit global MLP rather than hash-encoded feature grids, resulting in higher quality but significantly slower training and rendering speeds.

### [Neural Reconstruction in the Wild](6202-neural-reconstruction-in-the-wild.md)
Neural Reconstruction in the Wild is a neural radiance field method for reconstructing large-scale, real-world scenes. It addresses the low geometric reconstruction quality and coarse details of standard NeRF in expansive environments by using surface-guided sampling to improve reconstruction accuracy.

### [Point-NeRF](6569-point-nerf.md)
Point-NeRF is a point-based Neural Radiance Field representation that synthesizes novel views by aggregating features from a sparse 3D neural point cloud, avoiding empty space sampling by restricting shading locations to scene surfaces. It differs from the grid-based [NeRF](4972-neural-radiance-fields.md) by replacing dense volumetric grids with a learnable set of points, each carrying a high-dimensional feature vector, which allows for significantly faster training times (up to 30x) while maintaining comparable visual quality.

### [Volume Feature Rendering](6628-volume-feature-rendering.md)
Volume Feature Rendering (VFR) is a differentiable rendering technique that integrates queried feature vectors along a ray into a single aggregated feature vector, which is then transformed into the final pixel color by a single neural network evaluation. Unlike standard volume rendering, which requires $N$ neural network evaluations (one per sample) to predict color, VFR reduces the number of evaluations to one, thereby relieving the framework from the high computational cost associated with large color networks. This architectural change allows for the use of significantly larger neural networks to improve rendering quality without increasing training or rendering time proportionally.

### [Cascaded Bundle-Adjusting Neural Radiance Fields (CBARF)](8223-cascaded-bundle-adjusting-neural-radiance-fields-cbarf.md)
CBARF is a 3D reconstruction framework that simultaneously optimizes camera poses and scene representation from input images with imperfect or partially unknown camera poses. It addresses the limitation where standard Neural Radiance Fields (NeRF) and single-stage Bundle Adjustment (BA) methods fail or over-fit when initial pose deviations are large. Unlike single-stage BA which quickly falls into sub-optimal solutions, CBARF uses a coarse-to-fine multi-stage approach with cascaded compact BA modules to progressively refine poses, avoiding the learning of incorrect scene representations.

### [Fourier Features Mapping](8242-fourier-features-mapping.md)
Fourier Features Mapping (FFM) is a preprocessing technique that transforms low-dimensional input coordinates into a higher-dimensional space using sinusoidal functions, enabling fully-connected MLPs to learn high-frequency details that are otherwise inaccessible due to spectral bias. It sets itself apart from geometric approaches (like octrees or grid subdomains) by decoupling the neural network architecture from the spatial representation structure, allowing the same MLP to converge rapidly on sharp edges and fine textures without requiring architectural changes.

### [Reflection-aware NeRF](8310-reflection-aware-nerf.md)
Reflection-aware NeRF is a neural rendering method that jointly models planar surfaces and explicitly casts reflected rays to capture the source of high-frequency reflections, preventing false geometry duplication along the primary ray. It differs from methods like NeRFReN and MS-NeRF, which decompose the scene into independent components and may cause false geometries, by using a single radiance field and ensuring that if an actual object is observed along the reflected ray, no false duplicate is created along the primary ray at the same depth.

### [GRAF](8539-graf.md)
GRAF is a generative model for 3D-aware image synthesis that learns a conditional radiance field from a collection of unposed 2D images using an adversarial framework. It allows explicit control over camera rotation, translation, shape, and appearance by mapping latent codes and camera parameters to image patches via differentiable volume rendering. Unlike voxel-based approaches (e.g., HoloGAN, PLATONIC GAN) which suffer from memory limitations and discretization artifacts, GRAF uses a continuous representation that scales well with image resolution and incorporates 3D consistency by design without learned projections.

### [Flash Cache](9550-flash-cache.md)
Flash Cache is a volumetric inverse rendering method that reduces bias and variance in radiance estimation by using a high-capacity Neural Radiance Field (NeRF) as a control variate for incoming illumination. Unlike prior radiance caches that use low-capacity models, which fail to capture high-frequency and near-field illumination, Flash Cache uses a high-capacity cache. Unlike methods that use a NeRF as a cache but render at only a single evaluation point, which causes bias, Flash Cache uses the cache as a control variate for unbiased estimation. It achieves this by replacing the recursive incoming radiance term with a NeRF-based cache output, evaluated via importance sampling, while maintaining computational tractability through a "Fast Cache" approximation.

### [C-ToF Gaussian Splatting](9678-c-tof-gaussian-splatting.md)

### [Generative Neural Feature Fields (GIRAFFE)](10047-generative-neural-feature-fields-giraffe.md)
Generative Neural Feature Fields (GIRAFFE) is a scene representation framework that models individual objects within a scene using separate, compositional neural feature fields, enabling disentangled control over pose, shape, and appearance. Unlike its siblings [NeRF](1185-nerf-neural-radiance-fields.md) and [GRAF](8539-graf.md), which represent the entire scene with a single monolithic model, GIRAFFE assigns each object its own MLP that outputs a scalar density and a deep feature vector, composing the final scene via a density-weighted sum.

### [Fourier Feature Encoding](10580-fourier-feature-encoding.md)
Fourier Feature Encoding is a preprocessing technique applied to scene coordinates and viewing directions to address the spectral bias of neural networks, which naturally favor low-frequency functions. By mapping input vectors into a high-dimensional space via sinusoidal functions, it provides high-frequency features directly to the network, greatly improving fine detail reconstruction in rendered views. It sets itself apart from standard positional encodings (like those in Transformers) by using fixed, non-trainable sinusoidal bases rather than learned embeddings, and from learnable frequency mappings by relying on a deterministic geometric progression of frequencies.

### [Voxel-based Representation](10852-voxel-based-representation.md)
Voxel-based representations replace the coordinate-based Multi-Layer Perceptron (MLP) of standard Neural Radiance Fields with discrete 3D grids (voxels) to store scene features, enabling faster training and inference through spatial locality and convolutional operations. Unlike coordinate-based methods that evaluate a global function for every sample, voxel methods leverage dense or sparse tensor structures to capture spatial correlations, often using 3D convolutions for feature extraction. This family addresses the computational bottleneck of MLP evaluation and the lack of explicit spatial inductive biases in pure neural representations.

### [ExBluRF](11440-exblurf.md)
ExBluRF is a novel view synthesis method designed to reconstruct sharp 3D scenes from extreme motion-blurred images by jointly optimizing voxel-based radiance fields and 6-DOF camera trajectories. It distinguishes itself from sibling methods like [NeRF](1185-nerf-neural-radiance-fields.md) and [DeblurNeRF](6202-neural-reconstruction-in-the-wild.md) by replacing the memory-intensive MLP-based radiance field with a voxel grid and modeling motion blur as an accumulation of rays along a Bézier-curve-defined camera trajectory, rather than using 2D spatially varying blur kernels. This approach allows for a significantly higher number of sampling points along the trajectory without proportional increases in memory or computation, enabling the restoration of high-quality sharp views from severely degraded inputs.

### [Neural Volume Super-Resolution](11444-neural-volume-super-resolution.md)
Neural Volume Super-Resolution (NVSR) is a method that super-resolves volumetric neural scene representations to generate geometrically consistent, high-resolution novel views from low-resolution captures, operating directly on the 3D representation rather than post-rendered 2D images. It sets itself apart from post-rendering 2D SR methods by guaranteeing consistency across viewing directions and preventing inconsistent distortions, and from spatial Video SR methods by enabling novel views from arbitrary directions rather than being limited to the original camera trajectory.

### [Volumetric Supervision for NeRF](11452-volumetric-supervision-for-nerf.md)
Volumetric Supervision for NeRF is a hybrid reconstruction method designed to improve the fidelity of the canonical volume in dynamic tomography by combining projection-based training with volumetric supervision from high-fidelity static X-CT data. It addresses the limitation where neural rendering methods fail to achieve high normalized correlation coefficients for complex objects, even when conventional data-rich X-CT can reconstruct such volumes with high fidelity. It sets itself apart from standard NeRF optimization by explicitly incorporating a ground-truth volumetric loss term derived from external high-resolution scans, rather than relying solely on 2D projection consistency.

### [Part-based MHE-augmented NeRF](11665-part-based-mhe-augmented-nerf.md)
Part-based MHE-augmented NeRF is a method for learning neural volumetric videos of dynamic humans from sparse view videos in minutes with competitive visual quality. It addresses the challenge of quickly reconstructing free-viewpoint videos of dynamic humans by decomposing the canonical human body into multiple anatomical parts, assigning each part its own Multiresolution Hash Encoding (MHE)-augmented NeRF network to efficiently distribute representational power. Unlike a single-resolution representation that treats all body parts as equally complex, this method adapts the hash table size per part based on local geometric and texture complexity, accelerating convergence while maintaining high fidelity.

### [NeRF-MAE](11769-nerf-mae.md)
NeRF-MAE is a self-supervised pretraining framework for Neural Radiance Fields (NeRF) that learns 3D scene representations by masking and reconstructing patches from an explicit radiance and density grid. Unlike implicit NeRFs or point-cloud-based methods, it operates directly on the regular, high-density volumetric grid extracted from a trained NeRF, allowing the use of standard 3D Vision Transformers without adaptation. It disentangles representation learning from NeRF training, enabling the use of large-scale unlabeled data from diverse sources to improve downstream tasks such as 3D object detection and semantic labeling.

### [Distilled Feature Fields](12026-distilled-feature-fields.md)
Distilled Feature Fields (DFF) extend Neural Radiance Fields by adding a 3D neural feature field that maps every 3D coordinate to a semantic feature descriptor, enabling query-based local editing of NeRF scenes by semantically decomposing 3D space without re-training. Unlike methods that optimize NeRF parameters to match text prompts (which may affect unintended regions) or rely on costly instance segmentation annotations, DFF enables efficient, open-set decomposition and accurate selective editing of specific scene regions via decomposition. It builds on [Neural Radiance Fields](4972-neural-radiance-fields.md) by distilling knowledge from off-the-shelf 2D image feature extractors (teachers) into the 3D field via volume rendering.

### [NeuralRadianceFieldPackedRenderer](13115-neuralradiancefieldpackedrenderer.md)
A neural field renderer for pipelines combining NeuralRadianceField and PackedRFTracer. Registered as a default fallback for BaseNeuralField subclasses using PackedRFTracer. Capable of reducing tracer settings to lower quality during interactive mode.

### [Semi-Gradient Method](13620-semi-gradient-method.md)
A neural rendering optimization method that modifies the baseline loss by applying a stop-gradient operation to the RHS term in the numerator, thereby removing the partial derivatives with respect to the RHS to achieve unbiased and low-variance gradient estimates.

### [Hybrid Surface-Volume and Mesh Representation](13663-hybrid-surface-volume-and-mesh-representation.md)
This family represents scenes by combining explicit geometric primitives (surfels, meshes) or implicit surface fields with neural radiance fields, aiming to decouple geometry precision from appearance modeling [7358-neural-surfel-radiance-fields.md][9732-unified-neural-implicit-surface-and-radiance-field-rendering.md][12715-structured-latent-codes-anchored-to-mesh.md]. Unlike standard NeRF which relies solely on volumetric density for both structure and color, these methods use explicit surfaces or occupancy boundaries to guide sampling or define the radiance evaluation points, enabling faster rendering and higher geometric fidelity [7358-neural-surfel-radiance-fields.md][9732-unified-neural-implicit-surface-and-radiance-field-rendering.md].

### [Dynamic and Deformation Modeling](13664-dynamic-and-deformation-modeling.md)
Dynamic and Deformation Modeling extends static Neural Radiance Fields to reconstruct time-varying scenes from multi-view video or sparse image collections, solving the problem of novel view synthesis for non-static content. It sets itself apart from static NeRF variants by explicitly incorporating temporal information, either through direct time-conditioned radiance fields or by warping spatial coordinates to a canonical space, allowing for the coherent reconstruction of moving objects that would otherwise appear inconsistent in a static model.

## Background
Neural Radiance Fields (NeRF) were introduced by Mildenhall et al. in 2020 [S1, S3]. The method represents a continuous volumetric scene representation using a multi-layer perceptron (MLP) that maps 3D spatial coordinates and viewing directions to volume density and emitted color [S1, S3]. This approach solved the problem of high-quality novel view synthesis from a sparse set of 2D images by leveraging differentiable volume rendering to train the implicit representation [S1, S2]. Prior to NeRF, explicit representations like voxel grids or meshes required significant memory or manual geometry processing, while previous implicit methods lacked the capacity to model complex view-dependent effects [S1][S4, unofficial]. NeRF's use of positional encoding and hierarchical sampling allowed it to capture high-frequency details and realistic lighting, setting a new standard for neural rendering [S1, S3]. Since its introduction, the field has expanded into dynamic scenes, large-scale environments, and efficient training/inference techniques [S1, S2].

## Sources

[S1] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
[S2] NeRFPlayer: A Streamable Dynamic Scene Representation with Decomposed Neural Radiance Fields, https://doi.org/10.1109/tvcg.2023.3247082, fetched 2026-10-26
[S3] Nerfies: Deformable Neural Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00581, fetched 2026-10-26
[S4] Differentiable Rendering: A Survey, https://doi.org/10.48550/arxiv.2006.12057, fetched 2026-10-26 (unofficial)
[S5] Efficient View Synthesis with Neural Radiance Distribution Field, https://doi.org/10.1109/iccv51070.2023.01696, fetched 2026-10-26
[S6] HG3-NeRF: Hierarchical Geometric, Semantic, and Photometric Guided Neural Radiance Fields for Sparse View Inputs, https://doi.org/10.48550/arXiv.2401.11711, fetched 2026-10-29 (unofficial)
[S7] UVA: Towards Unified Volumetric Avatar for View Synthesis, Pose rendering, Geometry and Texture Editing, https://doi.org/10.48550/arXiv.2304.06969, fetched 2026-10-29 (unofficial)
[S8] HumanNeRF: Efficiently Generated Human Radiance Field from Sparse Inputs, https://doi.org/10.1109/cvpr52688.2022.00759, fetched 2026-10-26
