# Model Architecture Modifications

Also called: NeRF Architecture Variants, Radiance Field Network Modifications

## Overview
Model Architecture Modifications extend the standard Neural Radiance Field (NeRF) framework by altering the input conditioning, output heads, or underlying spatial representation to address specific limitations such as sparse views, dynamic scenes, or complex lighting. While the standard NeRF uses two Multi-Layer Perceptrons (MLPs) to map positional and directional encodings to density and color, this family encompasses methods that introduce auxiliary inputs (e.g., appearance embeddings, fluid volumes), auxiliary outputs (e.g., semantic labels, uncertainty), or substitute the continuous MLP with discrete structures (e.g., voxel grids, SDFs) [S6, unofficial]. These modifications enable capabilities ranging from depth-of-field simulation and real-time rendering to robust reconstruction from few-shot inputs and "in-the-wild" photo collections.

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
DoF-NeRF extends the standard NeRF to simulate depth-of-field effects by optimizing learnable aperture ($K$) and focus distance ($F$) parameters per view, using the Circle of Confusion (CoC) diameter to blur samples based on their distance from the focal plane. Choose this method when synthesizing all-in-focus views from shallow depth-of-field inputs or when explicit lens simulation is required.

### [NeRF-VPT](11770-nerf-vpt.md)
NeRF-VPT is a cascading view prompt tuning paradigm that uses RGB images from previous rendering stages as visual prompts for subsequent networks, progressively enhancing image quality without external networks. Choose this method for sparse-view synthesis where iterative refinement of internal network outputs improves scene understanding.

### [Deceptive-NeRF/3DGS](11859-deceptive-nerf-3dgs.md)
Deceptive-NeRF/3DGS mitigates overfitting in sparse-view reconstruction by treating photorealistic pseudo-observations generated via a 2D diffusion model as real training views, optimizing the radiance field on both real and synthetic data. Choose this method when training on very few inputs ($<20$) to avoid artifacts and accelerate training compared to per-step diffusion regularization.

### [MixRT](12401-mixrt.md)
MixRT is a hybrid representation combining a low-quality triangle mesh, a view-dependent displacement map, and a compressed NeRF model (Instant-NGP format) to enable real-time rendering on edge devices by offloading tasks to standard graphics hardware. Choose this method when real-time performance ($>30$ FPS) and low storage are critical constraints for large scenes.

### [Activation Feature Density Approximation](12492-activation-feature-density-approximation.md)
This method estimates volumetric density directly from intermediate activation features of the coarse MLP, replacing expensive fine network evaluations with lightweight statistical computations (mean and standard deviation of activations). Choose this method to accelerate inference by approximating density without evaluating the full fine network.

### [Signed Distance Function (SDF) Representations](12495-signed-distance-function-sdf-representations.md)
SDF-based Neural Fields replace the standard density output with a Signed Distance Function (SDF) value, modeling the distance to the nearest surface to provide explicit geometry and address surface holes typical of density-based NeRFs. Choose this method when precise surface extraction and smooth geometry are prioritized over volumetric appearance.

### [SDF-based Neural Fields](12947-sdf-based-neural-fields.md)
This variant models the distance to the nearest surface via an SDF, allowing for precise geometry extraction via the zero-level set, unlike standard NeRFs which model volume density $\sigma$. Choose this method for applications requiring explicit surface meshes or robust geometric constraints.

### [FlipNeRF](12967-flipnerf.md)
FlipNeRF improves few-shot novel view synthesis by generating auxiliary 'flipped reflection rays' from estimated surface normals, adding new training rays derived from geometric estimates rather than heuristic sampling. Choose this method for sparse-view reconstruction where surface normal accuracy and reflection handling are critical.

### [Volume-Augmented NeRF](12973-volume-augmented-nerf.md)
Volume-Augmented NeRF binds an explicit fluid volume value $v_i$ to each sample point, allowing the renderer to learn density and color based on the spatial distribution provided by a differentiable fluid simulator. Choose this method for dynamic scenes involving fluid dynamics where density changes are driven by physical flow.

### [Neural-PIL Decomposition Architecture](12975-neural-pil-decomposition-architecture.md)
This architecture separates scene geometry, material properties (BRDF embedding), and illumination (illumination embedding) into distinct networks, using a learned pre-integrated lighting network to render shaded colors. Choose this method when input images are captured under varying illumination and explicit reflectance decomposition is required.

### [Feature Fields](12995-feature-fields.md)
Feature Fields generalize the NeRF formulation by predicting arbitrary per-point feature signals $\mathbf{f}$ in addition to color and density, enabling applications like semantic segmentation and gaze redirection. Choose this method when the scene needs to encode semantic, physical, or latent information beyond appearance.

### [SAID-NeRF](13619-said-nerf.md)
SAID-NeRF exploits instance masks from Visual Foundation Models to enhance surface estimation of transparent objects by decomposing appearance into diffuse/specular components and enforcing higher surface densities via semantic segmentation. Choose this method for reconstructing transparent objects where RGB alone is ambiguous.

### [Differentiable Primitive Assembly](13657-differentiable-primitive-assembly.md)
Differentiable Primitive Assembly models 3D shapes as an assembly of quadric primitives using differentiable intersection and union operations, replacing the continuous density field with a discrete, structured representation. Choose this method when explicit shape abstraction and test-time adaptation of convex hulls are needed.

### [MINE](13661-mine.md)
MINE models the frustum of the source camera using a continuous depth Multiplane Image (MPI) representation, conditioned on input images via a fully-convolutional decoder, allowing rendering of any target view via homography warping. Choose this method for efficient novel view synthesis from a single source image, especially when rendering multiple target views.

### [IE-NeRF](13737-ie-nerf.md)
IE-NeRF separates static scene rendering from transient occlusions in "in-the-wild" scenes by using image-dependent feature encodings and a pre-trained inpainting module for supervision, rather than learning a 3D transient field. Choose this method for uncontrolled photo collections with moving objects or pedestrians.

### [Voxel and Sparse Grid Representations](13738-voxel-and-sparse-grid-representations.md)
This family replaces the continuous MLP with discrete data structures (dense grids, sparse grids, or hash tables) to store volume density and color features, enabling real-time rendering via direct interpolation. Choose this method when computational efficiency and real-time rendering are primary goals.

### [Aleth-NeRF](13764-aleth-nerf.md)
Aleth-NeRF trains on under- and over-exposed images to render normal-light novel views by introducing 'Concealing Fields' (local and global) that model illumination adaptive volume rendering. Choose this method when working with standard 8-bit LDR images captured in adverse lighting conditions.

### [Unified Volumetric Avatar (UVA)](13784-unified-volumetric-avatar-uva.md)
UVA represents dynamic human avatars by anchoring structured latent codes to a deformable mesh in canonical space, enabling local editing of geometry and texture while supporting novel view synthesis. Choose this method for dynamic human avatars requiring independent editing of shape and appearance.

### [Gear-NeRF](13848-gear-nerf.md)
Gear-NeRF leverages semantic information to stratify dynamic scenes into motion-based 'gears', assigning higher resolution sampling to regions with larger motion scales. Choose this method for free-viewpoint rendering of dynamic scenes with varying motion complexity.

### [Transformer and Vision Model Integration](13911-transformer-and-vision-model-integration.md)
This family integrates Vision Transformers (ViT) into NeRFs to leverage global semantic understanding and long-range dependencies, solving limitations of MLP-based NeRFs in single-image inputs or occlusions. Choose this method when generalizing across scenes or modeling self-occlusion via attention is required.

### [Human and Specific Subject Modeling](13997-human-and-specific-subject-modeling.md)
Methods in this family specialize NeRF architectures for human subjects, often incorporating skeletal priors or deformable meshes to handle non-rigid animation. Choose this method for high-fidelity human avatar reconstruction and animation.

### [OPONeRF](14014-oponerf.md)
OPONeRF personalizes network parameters for each sampled 3D point by dynamically conditioning weights on the point's representation and scene features, handling test-time scene changes and perturbations. Choose this method when robustness against local variations (motion, illumination) is critical.

### [Mixer-NeRF](14073-mixer-nerf.md)
Mixer-NeRF addresses blurry details in standard NeRFs by introducing a hybrid spatial feature learning module (MLCA) and a squeeze-and-excitation (SE) network module to mix spatial and channel features before MLP inference. Choose this method when high-frequency detail preservation is needed.

### [Lightning NeRF](14134-lightning-nerf.md)
Lightning NeRF is a hybrid representation for large-scale autonomous driving scenes, using explicit voxel grids for density (initialized with LiDAR) and implicit hash-based grids for color, decomposing the scene into foreground and background. Choose this method for large-scale outdoor scenes with LiDAR data.

### [SCALAR-NeRF](14267-scalar-nerf.md)
SCALAR-NeRF is a coarse-to-fine framework for large-scale reconstruction that decomposes scenes into spatial blocks, fusing local outputs using a reference image rendered from a coarse global model to avoid "foggy" artifacts. Choose this method for scalable reconstruction of expansive scenes on limited GPU memory.

### [Viewing Direction Gaussian Splatting](14273-viewing-direction-gaussian-splatting.md)
VDGS combines 3D Gaussian Splatting with NeRF by using a neural network to modulate opacity and color based on viewing direction, enabling view-dependent effects like specular reflections in a rasterization pipeline. Choose this method when view-dependent appearance is needed with the speed of Gaussian Splatting.

### [Sparse View and Few-Shot Optimization](14428-sparse-view-and-few-shot-optimization.md)
This family addresses NeRF degradation in sparse-view regimes ($|S| < 20$) by introducing in-situ priors, deformable sampling, or consistency regularization to stabilize training. Choose this method when training data is severely limited.

### [Mixture-of-Experts and Modular Architectures](14429-mixture-of-experts-and-modular-architectures.md)
These architectures replace the single continuous function of NeRF with multiple specialized networks (experts) or parallel streams to improve compression, training speed, or representation capacity. Choose this method for complex scenes requiring partitioned processing or boundary discontinuity reduction.

### [Uncertainty and Reliability](14487-uncertainty-and-reliability.md)
Uncertainty methods extend NeRF to quantify prediction confidence by modeling the scene as a probabilistic entity, providing pixel-level reliability metrics for color and depth. Choose this method for robust filtering and conformal prediction in sparse or unstructured data.

### [Object Decomposition and Composition](14518-object-decomposition-and-composition.md)
These methods explicitly separate a scene into distinct semantic or geometric object instances, enabling instance-level editing and amodal segmentation. Choose this method when structural decomposition for editing is the primary goal.

### [Lighting and Appearance Models](14522-lighting-and-appearance-models.md)
These models decouple scene geometry from illumination variations and transient artifacts using learned appearance embeddings or explicit pre-integrated lighting queries. Choose this method for real-world image collections with lighting changes and shadows.

### [Optimization and Training Procedures](14565-optimization-and-training-procedures.md)
This family covers algorithmic procedures for optimizing NeRFs, including structural modifications to the optimization scope such as skipping expensive evaluations or restricting updates. Choose this method for efficient minimization of photometric loss under computational constraints.

### [Image Enhancement and Restoration Methods](14566-image-enhancement-and-restoration-methods.md)
These methods modify the training pipeline to reconstruct sharp details from blurry or degraded inputs by explicitly modeling the degradation process during rendering. Choose this method when input images are low-resolution or defocused.

### [Modular, Hierarchical, and Latent Architectures](14567-modular-hierarchical-and-latent-architectures.md)
This family extends NeRF to handle non-rigid deformation with topological changes (HyperNeRF) or hierarchical scene structures (Strata-NeRF) using latent codes or higher-dimensional spaces. Choose this method for dynamic scenes with topological changes or stratified environments.

### [NeRF-W](14630-nerf-w.md)
NeRF-W handles unconstrained "in-the-wild" photo collections by disentangling shared static geometry from image-dependent transient elements and photometric variations using learned appearance embeddings and a secondary transient radiance field. Choose this method for reconstructing high-frequency details and occluders in dynamic, uncontrolled environments.

### GARF
GARF uses Gaussian activations as an effective alternative to positional encoding in NeRF, in conjunction with bundle adjustment for pose estimation from unknown camera poses [S1]. It successfully recovers scene representations from unknown camera poses in challenging scenes with low-textured regions.

### NeRF-VAE
NeRF-VAE is a generative NeRF model that generalizes to out-of-distribution scenes and removes the need to train on each scene from scratch by conditioning the NeRF renderer on a latent code trained using Iterative Amortized Inference and a ResNet encoder [S1]. It introduces an attention-based scene function instead of the typical MLP. It outperforms baseline NeRF with a low number of scene views (5-20) but is outperformed when a large number of views are available (100+) due to lower scene expressivity.

### PIE-NeRF
PIE-NeRF is a pipeline integrating physics-based elastodynamic simulation with a neural radiance field representation, using a meshless discretisation based on quadratic generalized moving least squares on top of the density field of a NeRF [S1]. It enables large deformations and nonlinear hyperelastic material simulation directly on the implicit representation and supports interactive manipulation of real-world captured scenes with complex geometries.

### DäRF Implementation Details
DäRF is implemented based on K-planes as NeRF, using DPT-hybrid as the MDE model [S2, unofficial]. The optimizer is Adam with a learning rate of $1 \cdot 10^{-2}$ for NeRF and $1 \cdot 10^{-5}$ for the MDE, along with a cosine warmup learning rate scheduling.

### NeRF Model Architecture
The standard neural network architecture for NeRF consists of dense layers with ReLU activations, potentially using view directions as input [S3, unofficial]. Default parameters include: `netdepth` (layers in network) = 8, `netwidth` (channels per layer) = 256, `netdepth_fine` = 8, `netwidth_fine` = 256, `output_ch` = 4 (RGB + density), and `use_viewdirs` = True.

### S3-NeRF
S3-NeRF uses shadow and shading cues to infer scene geometry and enable single-image NeRF training, with a focus on geometry recovery [S1]. It employs a UNISURF-based occupancy field 3D representation instead of density, a modified physics-based rendering equation, and occupancy-based shadow calculation. The method achieved excellent depth map and surface normal reconstruction from single images on both synthetic and real-world datasets.

### KiloNeRF-style Grid Distillation
KiloNeRF-style Grid Distillation distills a large teacher NeRF into a spatially partitioned grid of tiny MLPs to enable fast, adaptive evaluation by mapping 3D positions to local student networks [S4, unofficial]. It initializes a dictionary mapping 3D grid cell indices to TinyMLP instances and accumulates L2 loss between student and teacher outputs across all samples. Default parameters include: `grid_resolution` = (16,16,16), `num_distill_iterations` = 1000, `num_samples` = 384, `learning_rate` = 1e-3, and bounding box `bmin` = [0,0,0], `bmax` = [1,1,1].

### NeRF with Time Parameter
Adding a time parameter to the model inputs adapts NeRF for dynamic scenes by positionally encoding the time parameter and concatenating it with the positional encoding of spatial coordinates $\gamma_x(\mathbf{x})$ [S5]. While this improves quality over vanilla NeRF, it still lacks detail, suffers from blurry predictions, and fails to reason about scene dynamics.

## Background
The original NeRF architecture by Mildenhall et al. [1] introduced hierarchical rendering with coarse and fine networks to improve computational efficiency. Subsequent works have focused on improving geometry, training speed, and inference speed. SNeS (June 2022) [110] improved geometry by learning probable symmetries. S3-NeRF (October 2022) [111] used shadow and shading cues for single-image training. NeRF-VAE (January 2021) [136] proposed a generative model conditioned on latent codes. GARF (April 2022) [104] used Gaussian activations for pose estimation. PIE-NeRF (November 2023) [227] integrated physics-based simulation. FastSR-NeRF (December 2023) [228] introduced a super-resolution CNN. These developments highlight the evolution from static, dense MLP-based representations to more efficient, generative, and physically informed architectures.

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] DaRF: Boosting Radiance Fields from Sparse Inputs with Monocular Depth Adaptation, https://doi.org/10.48550/arxiv.2305.19201, fetched 2026-10-29 (unofficial)
[S3] YoujiaZhang/NeMF, https://github.com/YoujiaZhang/NeMF, fetched 2026-10-27 (unofficial)
[S4] mdmubarak/OctaNeRF, https://github.com/mdmubarak/OctaNeRF, fetched 2026-10-27 (unofficial)
[S5] Neural Scene Graphs for Dynamic Scenes, https://doi.org/10.1109/cvpr46437.2021.00288, fetched 2026-10-26
