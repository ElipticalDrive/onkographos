# Neural Radiance Fields

Also called: Explicit Radiance Field Representations

## Overview
Neural Radiance Fields (NeRF) represent a 3D scene as a continuous volumetric function parameterized by a neural network, mapping 3D spatial coordinates and 2D viewing directions to emitted RGB color and volume density [S1][S5, unofficial][S12, unofficial]. The family solves the problem of novel view synthesis by training this implicit representation to reproduce input views, enabling photorealistic rendering from sparse calibrated images [S1][S9, unofficial]. It distinguishes itself from earlier implicit methods by explicitly modeling view-dependent appearance (specularity, transparency) via an asymmetric MLP architecture and by using hierarchical sampling to efficiently resolve scene geometry [S2, unofficial][S8, unofficial].

## Shared principles
All methods in this family rely on the differentiability of the forward rendering operator and the propagation of gradients through the integration steps [44-radiance-field-optimization.md]. The core mechanism is the evaluation of the volume rendering equation and the computation of gradients with respect to scene parameters (density, geometry, transfer functions, camera poses) [44-radiance-field-optimization.md].

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
Represents a static 3D scene as a continuous volumetric function. Distinguishes itself by explicitly modeling view-dependent appearance via an asymmetric MLP and using hierarchical sampling. Choose for high-quality novel view synthesis of static scenes.

### [Neural Reconstruction in the Wild](6202-neural-reconstruction-in-the-wild.md)
Addresses large-scale real-world scenes. Sets itself apart by introducing surface-guided sampling to concentrate density near geometry, improving detail fidelity. Choose for large-scale environments where standard NeRF yields coarse details.

### [Point-NeRF](6569-point-nerf.md)
Synthesizes views by aggregating features from $K$ neighboring neural points in a 3D point cloud. Differs by replacing the continuous volumetric function with discrete neural points, adapting naturally to surfaces and accelerating training. Choose for faster training and explicit geometric priors.

### [Volume Feature Rendering](6628-volume-feature-rendering.md)
Integrates queried feature vectors along a ray into a single aggregated vector, transformed into color by a single network evaluation. Unlike standard volume rendering requiring $N$ evaluations, VFR reduces this to one, allowing larger networks without proportional cost. Choose for high-quality rendering with limited network evaluation budget.

### [DyNeRF](7274-dynerf.md)
Represents volumetric videos by conditioning a single shared MLP on spatial coordinates and a temporal embedding. Unlike methods using canonical space mappings, it treats the volume as a time-varying function directly. Choose for dynamic scene novel view synthesis without explicit deformation fields.

### [Neural Surfel Radiance Fields](7358-neural-surfel-radiance-fields.md)
Represents scenes as a dynamic collection of neural surfels, combining explicit geometry with neural appearance. Solves real-time large-scale reconstruction by replacing volume integration with rasterization-based rendering. Choose for online, real-time reconstruction of indoor scenes.

### [Cascaded Bundle-Adjusting Neural Radiance Fields (CBARF)](8223-cascaded-bundle-adjusting-neural-radiance-fields-cbarf.md)
Simultaneously optimizes camera poses and scene representation from images with imperfect poses. Uses a coarse-to-fine multi-stage approach with cascaded compact BA modules. Choose when initial camera poses are unknown or significantly erroneous.

### [Fourier Features Mapping](8242-fourier-features-mapping.md)
A coordinate encoding technique overcoming the spectral bias of vanilla MLPs. Transforms inputs to higher-dimensional vectors to capture high-frequency details. Choose to improve convergence and detail representation in implicit networks.

### [Reflection-aware NeRF](8310-reflection-aware-nerf.md)
Jointly models planar surfaces and casts reflected rays to capture high-frequency reflections. Prevents false geometry duplication by using a single radiance field for primary and reflected rays. Choose for scenes with significant planar reflections (mirrors, windows).

### [GRAF](8539-graf.md)
A generative model for 3D-aware image synthesis from unposed 2D images using an adversarial framework. Allows explicit control over camera pose, shape, and appearance via latent codes. Choose for generative 3D-aware synthesis from unposed image collections.

### [Flash Cache](9550-flash-cache.md)
Reduces bias and variance in radiance estimation by using a high-capacity NeRF as a control variate for incoming illumination. Provides an unbiased estimate via importance sampling. Choose for physically-based inverse rendering with complex lighting.

### [C-ToF Gaussian Splatting](9678-c-tof-gaussian-splatting.md)
Not given in the sources.

### [Unified Neural Implicit Surface and Radiance Field Rendering](9732-unified-neural-implicit-surface-and-radiance-field-rendering.md)
Combines implicit surface rendering with volume rendering for accurate surface reconstruction without per-pixel masks. Eliminates hierarchical sampling by parameterizing occupancy directly. Choose for precise surface reconstruction from multi-view RGB.

### [Neural Radiance Distribution Field](10216-neural-radiance-distribution-field.md)
Models parametric radiance distribution along a camera ray rather than sampling discrete points. Maps rays to Fourier coefficients for opacity and color. Choose for efficient rendering with smaller network capacities, balancing speed and quality.

### [Voxel-based Representation](10852-voxel-based-representation.md)
Replaces the continuous MLP with discrete 3D voxel grids to store features/density. Solves high computational cost by enabling explicit spatial caching. Choose for significantly faster rendering speeds at the cost of model compactness.

### [NeuMesh++](11394-neumesh.md)
A mesh-based neural radiance representation encoding disentangled geometry, texture, and semantic codes on mesh vertices. Enables efficient volumetric editing.

### [ExBluRF](11440-exblurf.md)
A novel view synthesis method for extreme motion blurred images, jointly optimizing sharp voxel-based radiance fields and 6-DOF camera trajectories.

### [Neural Volume Super-Resolution](11444-neural-volume-super-resolution.md)
Super-resolves volumetric neural scene representations to generate high-resolution novel views from low-resolution captures, operating directly on the 3D representation.

### [Incremental CONfidence (ICON)](11446-incremental-confidence-icon.md)
An optimization procedure for training NeRFs from 2D video frames that removes pose initialization requirements by assuming smooth motion and using a 'confidence' measure to reweight gradients.

### [Canonical Neural Radiance Field for X-ray Tomography](11451-canonical-neural-radiance-field-for-x-ray-tomography.md)
Encodes the 3D density field of a static object (canonical volume) using a NeRF, trained on rich projection data to serve as a prior for dynamic reconstruction.

### [Volumetric Supervision for NeRF](11452-volumetric-supervision-for-nerf.md)
Improves canonical volume quality in dynamic tomography by combining projection-based training with volumetric supervision from high-fidelity static X-CT data.

### [Part-based MHE-augmented NeRF](11665-part-based-mhe-augmented-nerf.md)
Decomposes the human body into anatomical parts, assigning each its own MHE-augmented NeRF network to distribute representational power and accelerate training.

### [NeRF-MAE](11769-nerf-mae.md)
A self-supervised pretraining framework using standard 3D Vision Transformers to learn 3D representations by masking and reconstructing patches from NeRF's radiance and density grid.

### Neural Rendering
**Definition:** Learn the rendering process from data instead of handcrafting differentiation. A scene generation network outputs a neural scene representation and a rendering network is jointly trained by minimizing image reconstruction error [S2, unofficial].
**Steps:** Improve neural rendering by adding inductive biases for 3D scenes and rendering into neural networks [S2, unofficial].
**Pitfalls:** Generalizing to scenes differing from training data, scaling to multiple objects, and human modification are limited [S2, unofficial].

### Generative Neural Feature Fields (GIRAFFE)
**Definition:** A method representing individual objects in a scene as separate neural feature fields, allowing for disentangled control of pose, shape, and appearance, unlike NeRF/GRAF which represent the entire scene with a single model [S3].
**Formula:** Positional encoding is applied element-wise to input components $t$ (from $\mathbf{x}$ or $\mathbf{d}$) with $L$ frequency octaves: $\gamma(t, L) = (\sin(2^0 t\pi), \cos(2^0 t\pi), \dots, \sin(2^L t\pi), \cos(2^L t\pi))$ [S3].
**Parameters:** $M_f$: The dimensionality of the feature vector $\mathbf{f}$ output by the object field, replacing the 3D RGB color output used in NeRF/GRAF [S3]. $\mathbf{z}_s, \mathbf{z}_a$: Latent codes sampled from a standard normal distribution $\mathcal{N}(0, I)$, representing shape and appearance respectively [S3].
**Layout:** Object Transformation Parameters $\mathbf{T}$: A set containing scale $\mathbf{s} \in \mathbb{R}^3$, translation $\mathbf{t} \in \mathbb{R}^3$, and rotation matrix $\mathbf{R} \in SO(3)$ [S3].

### NeRDF Learning via Knowledge Distillation
**Definition:** A training framework where a teacher NeRF synthesizes dense and diverse views that serve as pseudo-training data for the student NeRDF [S4].
**Steps:** Input ray encoding method that captures rich ray information is used [S4]. Online view sampling strategy is used to expand the diversity of the pseudo-training data [S4]. A volume density constraint loss is applied to promote the learning of a strong 3D prior [S4]. The distillation process includes an additional matching between the volume density distribution of the target NeRDF and the teacher NeRF [S4].

### Fourier Feature Encoding
**Definition:** A preprocessing step reformulating the network as $F_{\Theta} = F^{\prime}_{\Theta} \circ \gamma$, where $\gamma$ encodes inputs using Fourier features to provide high-frequency features, addressing the neural network bias toward low-frequency functions [S5, unofficial].
**Formula:** The positional encoding function $\gamma$ for vector $\mathbf{v}$ is defined as: $\gamma(\mathbf{v}) = \left( \sin\left(2^{0} \pi \mathbf{v}\right), \cos\left(2^{0} \pi \mathbf{v}\right), \ldots, \sin\left(2^{L-1} \pi \mathbf{v}\right), \cos\left(2^{L-1} \pi \mathbf{v}\right) \right)$, where $\mathbf{v} \in [-1, 1]$ and $L$ is the number of arbitrary features selected for encoding [S5, unofficial]. The final function composition is: $(\mathbf{c}, \sigma) = F^{\prime}_{\Theta}(\gamma(\mathbf{x}), \gamma(\mathbf{d}))$ [S5, unofficial].
**Parameters:** $L$: The number of arbitrary features selected for encoding in the Fourier feature encoding [S5, unofficial].

### Spline-based Deformation Field
**Definition:** A deformation field parametrization using cubic B-splines with time-modulated weights to map spatial coordinates $(x, y, z)$ at time $t$ to canonical coordinates $(x_0, y_0, z_0)$, enabling reconstruction from sparse projections [S6, unofficial].
**Steps:** Calculate the deformation field from sparse few-projection data recorded at intermediate times $t$ during the dynamic experiment [S6, unofficial]. Obtain full field spatio-temporal information by combining the canonical volume and the calculated deformation field [S6, unofficial].
**Pitfalls:** Deformation fields based on canonical models fail when the scene topology changes over time, as they assume topology-preserving motion [S6, unofficial].
**Comparison:** Unlike methods using general MLP architectures for deformation fields which require complex regularization like elastic energy, this method uses cubic B-splines to enable reconstruction from as few as two projections [S6, unofficial].

## Background
Neural Radiance Fields were introduced by Mildenhall et al. in 2020 [S1][S5, unofficial]. The method revolutionized novel view synthesis by demonstrating that a simple MLP, trained with volume rendering, could reconstruct complex 3D scenes from unstructured image collections. Prior to NeRF, implicit representations often struggled with high-frequency details due to the spectral bias of neural networks, a problem addressed by the introduction of positional encoding [S5, unofficial]. The family has since expanded to address limitations in dynamic scenes [S1], large-scale environments [S1], and generative modeling [S3], as well as efficiency improvements through voxel-based [S1] and distribution-based [S4] representations.

## Sources
[S1] NeRFPlayer: A Streamable Dynamic Scene Representation with Decomposed Neural Radiance Fields, https://doi.org/10.1109/tvcg.2023.3247082, fetched 2026-10-26
[S2] Differentiable Rendering: A Survey, https://doi.org/10.48550/arxiv.2006.12057, fetched 2026-10-26 (unofficial)
[S3] GIRAFFE: Representing Scenes as Compositional Generative Neural Feature Fields, https://github.com/chaerinkong/paper-reviews/issues/63, fetched 2026-10-28
[S4] Efficient View Synthesis with Neural Radiance Distribution Field, https://doi.org/10.1109/iccv51070.2023.01696, fetched 2026-10-26
[S5] Neural Radiance Fields - Lei Mao's Log Book, https://leimao.github.io/blog/Neural-Radiance-Fields-NeRF/, fetched 2026-10-27 (unofficial)
[S6] Neural rendering enables dynamic tomography, https://doi.org/10.48550/arxiv.2410.20558, fetched 2026-10-26 (unofficial)
