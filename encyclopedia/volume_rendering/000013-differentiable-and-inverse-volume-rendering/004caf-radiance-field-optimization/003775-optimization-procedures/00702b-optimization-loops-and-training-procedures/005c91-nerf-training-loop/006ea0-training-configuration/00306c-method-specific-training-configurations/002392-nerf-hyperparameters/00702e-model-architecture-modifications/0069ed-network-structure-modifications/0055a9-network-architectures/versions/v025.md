# Network Architectures
Also called: NeRF Network Architectures

## Overview
Network architectures in neural radiance fields modify the baseline continuous volumetric representation $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$ [G] to address specific constraints such as dynamic scenes [S5], noise [S1], semantic understanding [S1], or inference speed [S2]. These architectures alter the network topology, input space, or output fields to enable features like temporal consistency, object manipulation, or noise-aware rendering [S1, S2, S3][S4, unofficial][S5, S6].

## How it works
The baseline NeRF architecture defines a continuous volumetric scene representation $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$, where $\mathbf{x} \in \mathbb{R}^3$ is the spatial position, $\mathbf{d} \in \mathbb{R}^3$ is the viewing direction, $\sigma$ is the volume density, and $\mathbf{c} \in \mathbb{R}^3$ is the view-dependent color [G]. The rendering process integrates these outputs along rays using standard volume rendering equations [G].

Most modifications in this family retain the core volume rendering equation but alter one or more of the following components:
1. **Input Space:** Augmenting $\mathbf{x}$ and $\mathbf{d}$ with additional signals such as latent codes $\mathbf{z}$, time $t$, illumination embeddings $\mathbf{z}_l$, or external feature vectors $\mathbf{f}$ from vision models [S1].
2. **Network Topology:** Replacing the monolithic MLP with specialized branches (e.g., separate density and color networks), modular experts, or hybrid structures combining MLPs with convolutional or transformer backbones [S1].
3. **Output Space:** Extending the output tuple $(\sigma, \mathbf{c})$ to include additional fields such as uncertainty, semantic logits, or physical properties like albedo and normals [S1].

## Subtopics

### [MIMO-DONeRF](16217-mimo-donerf.md)
MIMO-DONeRF is a method that applies the Multi-Input Multi-Output (MIMO) neural rendering concept to the DONeRF architecture. It modifies the shading network to accept multiple spatial inputs and view directions simultaneously, producing multiple color and density outputs per forward pass. This approach improves inference speed compared to increasing the number of samples in standard DONeRF, while maintaining or improving image quality by allowing the network to accumulate neighbor information through grouped samples. It differs from standard DONeRF by processing $N_p$ samples in parallel rather than sequentially, and from MIMO-NeRF by being specifically adapted to the two-network structure (depth oracle and shading) of DONeRF.

### [Point-NeRF Adaptation for Object Manipulation](16395-point-nerf-adaptation-for-object-manipulation.md)
This method adapts the Point-NeRF architecture to enable 3D scene geometry decomposition and manipulation from 2D images. It solves the problem of editing specific objects within a complex scene by replacing the standard Multi-View Stereo (MVS) point cloud generation with ground-truth dense data during training and using a diffusion model to infer object codes for manipulation. It differs from standard Point-NeRF by integrating a DM-NeRF module to map spatial locations to object codes and modifying the rendering pipeline to support point transformation.

### [Sparse Latent Space for 4D Novel View Synthesis (SLS4D)](18112-sparse-latent-space-for-4d-novel-view-synthesis-sls4d.md)
SLS4D is a method for 4D novel-view synthesis that represents dynamic radiance fields using a learnable sparse latent space, reducing network parameters compared to dense grid discretizations while maintaining high-frequency detail via attention-based interpolation. It differs from dense grid methods (like TiNeuVox) by avoiding the $O(N^3TF)$ space complexity of 4D grids, and from per-image latent code methods by offering better generalization across time through continuous temporal interpolation.

### [Neural Feature Fusion Fields (N3F)](18114-neural-feature-fusion-fields-n3f.md)
Neural Feature Fusion Fields (N3F) is a method that distills dense 2D image features from a pre-trained teacher network into a 3D neural field (student) via differentiable volumetric rendering, enabling 3D-aware feature representation. Unlike Semantic NeRF or Panoptic NeRF which fuse discrete semantic labels, N3F fuses continuous dense feature vectors, allowing for open-world applications without manual labels. It differs from standard feature distillation because the student operates in 3D world space (ensuring spatial consistency) while the teacher operates in 2D image space.

### [CLNeRF Architecture with Embeddings](18159-clnerf-architecture-with-embeddings.md)
CLNeRF is a model architecture modification for NeRF that enables continual learning by incorporating scene changes (appearance and geometry) over time within a single model, preventing model size growth as new scans are added. It sets itself apart from sibling methods like MIMO-DONeRF and SLS4D by using explicit, trainable per-scan embedding vectors rather than latent space compression or multi-instance networks, allowing for efficient updates and low overhead for generative replay.

### [Mutual Information Gradient Shaping](18669-mutual-information-gradient-shaping.md)
Mutual Information Gradient Shaping (MIGS), implemented as JacobiNeRF, is a training regularization technique that shapes the tangent space of a Neural Radiance Field (NeRF) to encode semantic correlations between scene entities. Unlike first-order approaches that distill 2D features into 3D radiance channels or treat labels as separate output channels, MIGS aligns the Jacobians of correlated entities to maximize their mutual information under random weight perturbations, enabling efficient sparse label propagation and semantic consensus.

### [Multi-Level Neural Scene Graph](19889-multi-level-neural-scene-graph.md)
Multi-Level Neural Scene Graph (NSG) is a scene representation that decomposes an environment into a graph of static, dynamic, and temporal nodes to model urban environments with moving objects and appearance variations. It distinguishes itself from sibling architectures by explicitly separating the radiance field into a static component $\phi$ and a dynamic component $\psi$, conditioned on sequence-level and object-level latent vectors respectively, enabling the disentanglement of scene-dependent appearance from object texture.

### [NAN (Noise-Aware NeRFs)](20279-nan-noise-aware-nerfs.md)
NAN is a burst-denoising method that extends the IBRNet architecture to handle degraded noisy images, such as those captured in low-light conditions, by modifying feature extraction, aggregation, and color blending to be noise-aware. Unlike standard NeRFs that synthesize novel views, NAN fixes an existing view image by leveraging multiple noisy inputs of the same scene. It distinguishes itself from its sibling MIMO-DONeRF and Point-NeRF Adaptation for Object Manipulation by explicitly modeling spatial noise statistics through 3D blending kernels and cross-view variance features, rather than relying solely on single-pixel cross-view consistency.

### [Multiplicative Interaction Module](21736-multiplicative-interaction-module.md)
The Multiplicative Interaction Module (MIM) is a neural component that generates conditional inputs for a NeRF by computing multiplicative interactions between expression, identity, and optional latent codes to improve facial disentanglement and rendering quality. It sets itself apart from sibling methods like MIMO-DONeRF or Point-NeRF Adaptation for Object Manipulation by explicitly modeling higher-order (second and third) multiplicative dependencies between semantic codes rather than relying on linear concatenation or simple additive fusion.

### [Part-Wise NARF (NARFP)](23078-part-wise-narf-narfp.md)
Part-Wise NARF (NARFP) is a neural articulated radiance field method that models an articulated object by training a separate Ray-Traced NeRF (RT-NeRF) for each rigid part, combining their outputs to estimate global density and color. It differs from holistic approaches by explicitly computing rigid transformations via forward kinematics, thereby avoiding implicit transformation issues and part dependency problems inherent in other methods.

### Dynamic Scene Network Architecture
This architecture divides dynamic scenes into static and dynamic parts modeled by two separate MLPs [S5].
**Inputs:** Static Network $F_{\Theta}^{st}$ takes positional encoding of position $\gamma(\mathbf{x})$ and direction $\gamma(\mathbf{d})$ [S5]. Dynamic Network $F_{\Theta}^{dy}$ takes positional encoding of position $\gamma(\mathbf{x})$, direction $\gamma(\mathbf{d})$, and time $\gamma(t)$ [S5].
**Outputs:** Static Network outputs color $\mathbf{c}$, density $\sigma$, and blending weight $v$ [S5]. Dynamic Network outputs color $\mathbf{c}_t$, density $\sigma_t$, scene flow $\mathcal{F}_t$, and disocclusion weights $\mathcal{W}_t$ [S5].
**Formulas:**
$$(\mathbf{c}_t, \sigma_t, \mathcal{F}_t, \mathcal{W}_t) = F_{\Theta}^{dy}(\gamma(\mathbf{x}), \gamma(\mathbf{d}), \gamma(t)) \quad (22)$$
$$(\mathbf{c}, \sigma, v) = F_{\Theta}^{st}(\gamma(\mathbf{x}), \gamma(\mathbf{d})) \quad (23)$$
where $\gamma(\cdot)$ denotes the positional encoding [S5].
**Steps:** Use equidistant sampling to obtain spatial points along rays, instead of hierarchical sampling [S5]. Use predicted scene flow to establish inter-frame relationship for time consistency [S5]. Jointly optimize camera parameters $\Delta f, \Delta P$ along with the MLPs $F_{\Theta}^{dy}, F_{\Theta}^{st}$ [S5].

### NeRF Network Architecture with Blend Weight Field
This architecture extends the radiance field with a blend weight field $F_{\Delta w}$ to handle dynamic human bodies [S6].
**Parameters:** Sampling uses single-level NeRF with 64 points along each camera ray [S6]. Appearance code and blend weight field code dimensions are both 128 [S6].
**Network Layout:** Radiance fields $F_\sigma, F_c$ follow the original NeRF architecture [S6]. Blend weight field $F_{\Delta w}$ is almost identical to $F_\sigma$, except the final output layer has 24 channels and applies $\exp(\cdot)$ to the output [S6].
**Steps:** Train parameters of $F_\sigma, F_c, F_{\Delta w}$, appearance codes $\{\mathbf{l}_i\}$ and blend weight field codes $\{\boldsymbol{\psi}_i\}$ jointly over the input video [S6]. Learn neural blend weight fields under novel human poses using equation (9) from the source [S6]. Use Adam optimizer with learning rate starting from $5 \times 10^{-4}$ decaying exponentially to $5 \times 10^{-5}$ [S6].

### OccNet
Occupancy Networks is a single-level version of NeRF that does not leverage positional encodings [S4, unofficial].
**Performance:** 2D Images: 13.90 PSNR / 0.437 SSIM [S4, unofficial]. Performance on Radiance Field: 20.60 PSNR / 0.849 SSIM [S4, unofficial]. Training Time: 02:17 / 100:23 (2D / RF). Model Size: 0.38 / 0.43 (2D / RF) [S4, unofficial].

### Residual-NeRF-S
An ablation variant of Residual-NeRF where background and residual NeRFs are merged by simply adding their outputs before activation, instead of using a Mixnet [S3].
**Formulas:**
$$\hat{\mathbf{C}}(\mathbf{r}) = \sum_{i=1}^{N} T_i (1 - \exp(-(\sigma_{bg} + \sigma_{res})_i \delta_i)) (\mathbf{c}_{bg+res})_i \quad (8)$$
with
$$\mathbf{c}_{bg+res} = S(\mathbf{c}'_{bg} + \mathbf{c}'_{res})$$
$$(\mathbf{c}'_{bg}, \sigma_{bg}) = \Phi_{bg}(h(\mathbf{x}, \mathbf{y}, \mathbf{z}, \theta, \phi))$$
$$(\mathbf{c}'_{res}, \sigma_{res}) = \Phi_{res}(h(\mathbf{x}, \mathbf{y}, \mathbf{z}, \theta, \phi))$$
$$S(\mathbf{x}) = \frac{1}{1 + e^{-\mathbf{x}}}$$
where $S(\cdot)$ is the sigmoid activation [S3].
**Comparison:** Residual-NeRF-S produces worse depth and view reconstruction compared to Residual-NeRF with Mixnet, especially for partially opaque objects [S3].

### Pyramid NeRF
Pyramid NeRF accomplishes rapid NeRF training and inference through the application of an image pyramid and a strategy based on frequency optimization [S2]. This technique utilizes a "low-frequency-first, high-frequency-second" approach, substituting large MLPs with thousands of smaller ones via progressive segmentation, markedly enhancing both training and inference efficiency [S2]. Furthermore, the introduction of a sampling approach that is aware of depth aims to enhance the quality of rendering and hasten the convergence of training [S2]. The experimental findings reveal that Pyramid NeRF enhances training and inference speeds by 15x and 805x, respectively, and notably betters structural and perceptual resemblance relative to the original NeRF [S2].

## Operating characteristics
* **Dynamic Scene Network:** Uses equidistant sampling instead of hierarchical sampling, which may affect convergence speed but simplifies the training pipeline for dynamic scenes [S5].
* **Pyramid NeRF:** Enhances training speed by 15x and inference speed by 805x relative to the original NeRF [S2].
* **OccNet:** Performance on 2D Images: 13.90 PSNR / 0.437 SSIM [S4, unofficial]. Performance on Radiance Field: 20.60 PSNR / 0.849 SSIM [S4, unofficial]. Training Time: 02:17 / 100:23. Model Size: 0.38 / 0.43 [S4, unofficial].
* **Residual-NeRF-S:** Produces worse depth and view reconstruction compared to Residual-NeRF with Mixnet, especially for partially opaque objects [S3].
* **Blend Weight Field:** Training takes around 200k iterations (about 12 hours) for a three-view video of 300 frames on four 2080 Ti GPUs [S6].

## Background
Neural Radiance Fields (NeRF) were introduced by Mildenhall et al. in 2020, establishing the baseline continuous volumetric representation [G]. Subsequent research has modified this architecture to handle specific challenges such as dynamic scenes [S5], noise [S1], semantic understanding [S1], and efficient inference [S2]. The modifications described here represent the evolution of the NeRF architecture from a static, monolithic MLP to specialized, multi-component systems [S1, S2, S5, S6].

## Sources
[S1] MobileNeRF: Exploiting the Polygon Rasterization Pipeline for Efficient Neural Field Rendering on Mobile Architectures, https://doi.org/10.1109/cvpr52729.2023.01590, fetched 2026-10-26
[S2] Neural Radiance Field-based Visual Rendering: a Comprehensive Review, https://ieeexplore.ieee.org/abstract/document/11455380, fetched 2026-10-30
[S3] Residual-NeRF: Learning Residual NeRFs for Transparent Object Manipulation, https://doi.org/10.1109/icra57147.2024.10611224, fetched 2026-10-26
[S4] Factor Fields: A Unified Framework for Neural Fields and Beyond, https://doi.org/10.48550/arxiv.2302.01226, fetched 2026-10-26 (unofficial)
[S5] Dynamic View Synthesis From Small Camera Motion Videos, https://doi.org/10.1109/tvcg.2025.3565642, fetched 2026-10-27
[S6] Animatable Neural Radiance Fields for Modeling Dynamic Human Bodies, https://doi.org/10.1109/iccv48922.2021.01405, fetched 2026-10-26
