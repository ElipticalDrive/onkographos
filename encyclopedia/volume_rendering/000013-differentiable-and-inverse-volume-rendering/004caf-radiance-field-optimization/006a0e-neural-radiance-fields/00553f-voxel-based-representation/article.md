# Voxel-based Representation

Also called: Voxel Grids, 3D Grids, Discrete Radiance Fields

## Overview
Voxel-based representations use discrete 3D grids to store volumetric features, density, or color [S1]. This approach trades spatial continuity for memory efficiency and inference speed, with resolution limits imposed by grid size [S1].

## Shared principles
The family builds on the differentiability of the forward rendering operator and the propagation of gradients through the integration steps [G]. The core mechanism is the evaluation of the volume rendering equation and the computation of gradients with respect to scene parameters (density, geometry, transfer functions, camera poses) [G].

The standard emission-absorption model computes the color $\hat{C}(\mathbf{r})$ for a ray $\mathbf{r}$ as:

$$
\hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
$$

where $\sigma_i$ is the density, $\delta_i$ is the step size, $\mathbf{c}_i$ is the color, and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the transmittance [G].

## Subtopics

### [FastNeRF](7183-fastnerf.md)
FastNeRF is a neural rendering method that accelerates inference by factorizing the view-dependent radiance field into position-dependent and direction-dependent components, enabling efficient caching of intermediate network outputs. It solves the problem of high computational cost in standard NeRF by avoiding MLP evaluations at inference time, instead relying on hardware-accelerated ray tracing and grid lookups to achieve interactive frame rates (200Hz) on consumer hardware. It differs from its siblings, such as [C-NeRF](7275-c-nerf.md) and [V4D](8762-v4d-voxel-for-4d-novel-view-synthesis.md), by using a low-rank inner-product factorization to decouple spatial and directional information, whereas other methods may use spherical harmonics or explicit voxel grids for density only.

### [C-NeRF](7275-c-nerf.md)
C-NeRF is a voxel-based radiance field representation that uses a 3D convolutional neural network (CNN) to generate feature volumes, which are then decoded by a NeRF-like multi-layer perceptron (MLP) to predict density and color. C-NeRF serves as a baseline for dynamic scene representation. When decomposing 4D volumes into a weighted summation of shared 3D volumes, this factorization is referred to as Volume Basis.

### [V4D (Voxel for 4D Novel View Synthesis)](8762-v4d-voxel-for-4d-novel-view-synthesis.md)
V4D is a voxel-based architecture for 4D novel view synthesis in dynamic scenes that models the 4D neural radiance field directly without requiring a canonical space. It distinguishes itself from continuous neural radiance fields and other voxel-based methods by using separate 3D voxel grids for density and texture, combined with a conditional positional encoding (CPE) module and a look-up table (LUT) refinement module to mitigate the over-smoothing effects of total variation regularization. Unlike its sibling [TiNeuVox](12633-tineuvox.md) which uses a canonical space and position offsets, V4D relies on temporal concatenation and total variation loss to enforce temporal consistency, resulting in longer training times but faster inference.

### [RenderNet](10878-rendernet.md)
RenderNet is a differentiable rendering convolutional neural network that generates 2D images from 3D voxel grids by jointly learning visibility computation and pixel color calculation (shading). Unlike traditional rasterization or ray tracing which rely on discrete, non-differentiable operations like z-buffering, RenderNet uses a CNN to encode spatial occlusion and shading automatically. It is distinguished from sibling methods like [FastNeRF](7183-fastnerf.md) and [VoxGRAF](12633-voxgraf.md) by its explicit use of a "Projection Unit" that collapses the 3D depth dimension into 2D feature maps while learning visibility, rather than relying on continuous volume rendering equations or sparse voxel structures. It handles low-resolution or noisy voxel inputs more robustly than mesh-based differentiable renderers.

### [ExBluRF](11440-exblurf.md)
ExBluRF is a novel view synthesis method designed to reconstruct sharp 3D scenes from extreme motion-blurred images by jointly optimizing voxel-based radiance fields and 6-DOF camera trajectories. It distinguishes itself from sibling methods like [FastNeRF](7183-fastnerf.md) and [C-NeRF](7275-c-nerf.md), which focus on static scene acceleration, by addressing the ill-posed problem of deblurring through a trajectory-based blur model and memory-efficient voxel representations, unlike [DeblurNeRF](7183-fastnerf.md) which relies on spatially varying 2D kernels and MLPs.

### [Part-based MHE-augmented NeRF](11665-part-based-mhe-augmented-nerf.md)
Part-based MHE-augmented NeRF is a method for learning neural volumetric videos of dynamic humans from sparse view videos in minutes with competitive visual quality. It addresses the slow convergence of per-scene optimization methods by decomposing the canonical human body into multiple anatomical parts, assigning each part its own Multiresolution Hash Encoding (MHE)-augmented NeRF network with varying resolutions to efficiently distribute representational power. Unlike single-resolution voxelized representations or continuous implicit functions that treat the entire body uniformly, this method leverages a 2D motion parameterization scheme on a parametric mesh (SMPL) to accelerate deformation field learning.

### [Object-NeRF](11987-object-nerf.md)
Object-NeRF is a composition model that separates and edits objects within a scene using a hybrid voxel-based approach with jointly optimized voxel grid features and MLP parameters. It differs from its siblings by explicitly partitioning the scene into a background "scene branch" and foreground "object branch," allowing for the shifting, addition, or removal of specific objects without retraining the entire radiance field.

### [VoxGRAF](12633-voxgraf.md)
VoxGRAF is a 3D-aware Generative Adversarial Network (GAN) that replaces the coordinate-based Multi-Layer Perceptron (MLP) scene representation with a sparse voxel grid generator. It solves the problem of slow rendering and viewpoint/content entanglement inherent in previous 3D-aware GANs by enabling fast, 3D-consistent image synthesis with a single forward pass. Unlike its siblings [FastNeRF](7183-fastnerf.md), [C-NeRF](7275-c-nerf.md), [V4D](8762-v4d-voxel-for-4d-novel-view-synthesis.md), [RenderNet](10878-rendernet.md), and [Object-NeRF](11987-object-nerf.md) which often rely on dense coordinate-based queries or hybrid rendering, VoxGRAF generates the entire 3D radiance field on a sparse voxel grid using 3D convolutions, allowing for efficient rendering via alpha composition without reducing sample counts.

### [Voxel-based Gaussian Splatting](13071-voxel-based-gaussian-splatting.md)
Voxel-based Gaussian Splatting (VoxGS) is a volume representation that places isometric 3D Gaussians at the fixed centroids of a regular 3D voxel grid to enable efficient Computed Tomography (CT) volume extraction and differentiable X-ray rendering. Unlike standard 3D Gaussian Splatting (3DGS) which optimizes primitive positions and uses spherical harmonics for view-dependent color, VoxGS fixes positions and removes color coefficients, retaining only opacity, scale, and rotation to support isotropic density modeling. This design solves the instability and inaccurate volume extraction issues associated with applying original 3DGS directly to CT data.

### [Gaussian Voxel Kernel Functions (GVKF)](13979-gaussian-voxel-kernel-functions-gvkf.md)
Gaussian Voxel Kernel Functions (GVKF) is a method for efficient 3D surface reconstruction in open scenes that establishes a continuous scene representation based on discrete 3D Gaussian Splatting (3DGS) through kernel regression. It differs from explicit 3DGS by using implicit continuous representation via kernel regression instead of explicit discrete summation, reducing memory consumption and improving surface continuity. Unlike standard volume rendering which relies on continuous sampling, GVKF integrates fast 3DGS rasterization with implicit scene representations to achieve high-fidelity open scene surface reconstruction without computationally intensive volume rendering.

### [3D Gaussian Particle Approximation of VDB Datasets](14723-3d-gaussian-particle-approximation-of-vdb-datasets.md)
This method converts OpenVDB volumetric datasets into a set of non-overlapping 3D Gaussian particles to enable memory-efficient, lossy representation with Level of Detail (LOD) support, rendered via forward ray marching using NVIDIA OptiX. It differs from standard voxel-based representations by replacing the discrete grid sampling with continuous Gaussian primitives, allowing for hardware-accelerated bounding box intersections and real-time performance with millions of particles, while maintaining compatibility with traditional volume rendering transfer functions.

### [Global Scene Generator (GSN)](16709-global-scene-generator-gsn.md)
Global Scene Generator (GSN) is a generative model that synthesizes unconstrained 3D scenes by decomposing them into local regions using a global generator and a locally conditioned radiance field. Unlike dictionary-based memory models that limit unseen viewpoints, GSN uses a learned scene prior to allow extrapolation beyond given observations. It sets itself apart from sibling voxel-based methods by explicitly generating a 2D latent "floorplan" grid to condition multiple local radiance fields, rather than storing static features in a 3D voxel grid.

### Neural Voxel Renderer (NVR)
- **Definition:** A framework that synthesizes realistic images given object voxels as input, providing editing functionalities for illumination, object geometry, and appearance [S1].
- **Parameters:** Initial voxel resolution is $100^3$. Scene volume is $V \in 128^3 \times 4$. Environment map resolution (realism variant) is $32 \times 32$, processed by convolutional layers to extract a latent code supplied to NVR+ [S1].
- **Performance:** At $25^3$ voxel resolution, performance decreases gracefully but still produces plausible outputs [S1].
- **Comparison:** NVR captures color and structure but smooths fine texture details. NVR+ accurately renders geometry and texture, realistically synthesizing shadows, specular reflections, and highlights [S1].
- **Usage:** Trained on ShapeNet chair category. Tested on Pix3D and Ikea datasets for sofas, tables, and miscellaneous objects [S1].

## Background
Not given in the sources.

## Sources
[S1] Neural Voxel Renderer: Learning an Accurate and Controllable Rendering Tool, https://doi.org/10.1109/cvpr42600.2020.00546, fetched 2026-10-26
