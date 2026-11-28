# HybridNeRF Model Architecture


## Overview
HybridNeRF is a neural radiance field architecture that combines multi-resolution 3D feature grids with 2D triplanes to efficiently represent complex scenes [S1, S2]. It solves the trade-off between the high memory cost of pure 3D voxel representations and the limited geometric expressiveness of pure 2D plane-based methods by using a hybrid query mechanism [S1]. Unlike its sibling [MVS Network Query](7517-mvs-network-query.md) which relies on multi-view stereo consistency, or [VoxNeRF](8865-voxnerf.md) which uses pure voxel grids, HybridNeRF explicitly separates geometric and appearance features into distinct multi-resolution structures [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D point coordinate $\mathbf{x} \in \mathbb{R}^3$ and a viewing direction $\mathbf{d} \in \mathbb{R}^3$.
*   **Output:** Volume density $\sigma \in \mathbb{R}$ and color $\mathbf{c} \in \mathbb{R}^3$ [S1].

**Data Structures:**
1.  **3D Feature Grids:** 3 resolution levels at $128^3$, $256^3$, and $512^3$ [S2].
2.  **Triplanes:** 7 resolution levels ranging from $128^2$ to $8,192^2$ [S2].
3.  **Hash Tables:** Length $T = 2^{19}$, feature dimension $F = 2$ per level [S1].
4.  **MLPs:** Density network $\mathcal{F}_{\mathbf{w}_\theta}$ (1 hidden layer, 64 channels) and color network $\mathcal{F}_{\mathbf{w}_\phi}$ (2 hidden layers, 64 channels each) [S1].

**Algorithm Steps:**
1.  **Coarse-level encoding:** Compute $\gamma_{\text{coarse}}(\mathbf{x}; \boldsymbol{\alpha}) = \gamma_p(\mathbf{x}) \otimes \boldsymbol{\alpha}$, where $\gamma_p(\mathbf{x})$ is fixed positional encoding with $L_c=8$ frequency bands, and $\boldsymbol{\alpha}$ are learnable weights [S1].
2.  **Fine-level encoding:** Query multi-resolution hash grids $\gamma_{\text{fine}}(\mathbf{x}; \boldsymbol{\vartheta})$ at $L_f \in \{8, 16\}$ levels, with resolution $N_l = \lfloor N_{\min} b^l \rfloor$ and $b = \exp((\ln N_{\max} - \ln N_{\min}) / (L_f - 1))$, where $N_{\min}=180$ [S1].
3.  **Cone tracing features:** Compute covariance $\mathbf{\Sigma} = \sigma_t^2(\mathbf{d}\mathbf{d}^T) + \sigma_r^2(\mathbf{I} - \mathbf{d}\mathbf{d}^T/\|\mathbf{d}\|_2^2)$, extract upper triangular elements $f(\mathbf{x}) = \text{triu}(\mathbf{\Sigma})$, and encode via $\gamma_p$ [S1].
4.  **Learnable weights:** Predict $\boldsymbol{\alpha} = \mathcal{F}_{\mathbf{w}_p}([\gamma_{\text{fine}}(\mathbf{x}; \boldsymbol{\vartheta}), \gamma_p(f(\mathbf{x}))])$ [S1].
5.  **Hybrid position encoding:** Concatenate coarse and fine encodings: $\gamma_{\text{hyb}}(\mathbf{x}; \boldsymbol{\alpha}, \boldsymbol{\vartheta}) = [\gamma_{\text{coarse}}(\mathbf{x}; \boldsymbol{\alpha}), \gamma_{\text{fine}}(\mathbf{x}; \boldsymbol{\vartheta})]$ [S1].
6.  **Direction encoding:** Transform $\mathbf{d}$ into spherical harmonic features $\xi(\mathbf{d})$ up to the 4th degree [S2].
7.  **Feature aggregation:** Fetch $K=4$ features per level from 3D grids and triplanes via interpolation, sum across levels separately, and concatenate to obtain a $4K=16$-dimensional input vector [S1, S2].
8.  **MLP prediction:** Pass aggregated features through $\mathcal{F}_{\mathbf{w}_\theta}$ to get density $\sigma$ and intermediate features $\mathbf{e}$, then through $\mathcal{F}_{\mathbf{w}_\phi}$ with $\xi(\mathbf{d})$ to get color $\mathbf{c}$ [S1].
9.  **Volume rendering:** Compute pixel color $\hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i$ with transmittance $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ [S1].
10. **Optimization:** Minimize $\mathcal{L} = \|\mathbf{C}(\mathbf{r}) - \hat{C}(\mathbf{r})\|_2^2$ using Adam ($\beta_1=0.9, \beta_2=0.99$, lr=$5\times10^{-3}$, batch size 256K) [S1].

**Parameters:**
*   $L_c = 8$ frequency bands [S1].
*   $N_{\min} = 180$ coarsest resolution [S1].
*   $L_f \in \{8, 16\}$ fine-level hash levels [S1].
*   $T = 2^{19}$ hash table length, $F = 2$ feature dimension [S1].

**Pitfalls:**
*   **Aliasing:** Multi-resolution grids can introduce aliasing artifacts; this is mitigated by the hybrid encoding design and cone tracing features [S1, S2].
*   **Boundary handling:** Standard NeRF volume rendering bounds apply; out-of-bounds queries must be clamped or masked [G].
*   **Precision:** Fixed positional encoding at coarse levels guarantees convergence, while trainable fine-level grids capture high-frequency details [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hybrid_nerf_forward(x, d, grids_3d, triplanes_2d, params):
    # 1. Fixed positional encoding (L_c=8)
    gamma_p = fixed_positional_encoding(x, L_c=8)
    
    # 2. Fine-level hash features (L_f=8 or 16)
    gamma_fine = fetch_hash_features(x, grids_3d, levels=params.L_f)
    
    # 3. Cone tracing features
    Sigma = params.sigma_t**2 * outer(d, d) + params.sigma_r**2 * (eye(3) - outer(d, d) / norm(d)**2)
    f_x = upper_triangular(Sigma)
    gamma_p_cone = fixed_positional_encoding(f_x, L_c=8)
    
    # 4. Learnable weights alpha
    alpha = mlp_weight_predictor(concat(gamma_fine, gamma_p_cone))
    
    # 5. Coarse encoding with alpha
    gamma_coarse = elementwise_multiply(gamma_p, alpha)
    
    # 6. Hybrid position encoding
    gamma_hyb = concat(gamma_coarse, gamma_fine)
    
    # 7. Direction encoding (SH degree 4)
    d_sh = spherical_harmonics(d, degree=4)
    
    # 8. Feature aggregation (3D + 2D)
    f_3d = sum_features(fetch_3d_features(x, grids_3d, K=4))
    f_2d = sum_features(fetch_2d_features(x, triplanes_2d, K=4))
    f_total = concat(f_3d, f_2d)  # 16-dimensional per evidence [S1, S2]
    
    # 9. MLPs
    sigma, e = mlp_density(f_total)
    c = mlp_color(concat(e, d_sh))
    
    return sigma, c
```

## Subtopics

### [SeaThru-NeRF](11884-seathru-nerf.md)
A neural radiance field method for visualizing scenes in dispersed scattering media like fog or subaquatic environments. It extends standard NeRF by treating the scattering medium as a semi-transparent entity with distinct color and density parameters, enabling restoration of scene hue and 3D configuration in obscured areas. It builds on HybridNeRF by incorporating specific scattering physics into the volume rendering integral.

### [FastSR-NeRF](16574-fastsr-nerf.md)
Improves rendering efficiency on consumer hardware by combining a lightweight NeRF with a CNN-based super-resolution module. It trains the NeRF to produce low-resolution 3D-consistent features and upscales them to high-resolution outputs, requiring no architectural changes to the NeRF backbone.

### [NeRFCodec](16590-nerfcodec.md)
An end-to-end compression framework for plane-based hybrid NeRFs that integrates non-linear transform, quantization, and entropy coding. It compresses feature planes using a pre-trained 2D neural image codec backbone, achieving high-quality novel view synthesis within a strict memory budget of approximately 0.5 MB per scene.

### [LU-NeRF](17657-lu-nerf.md)
A few-shot local unposed NeRF method that jointly estimates camera poses and scene representation from a small subset of images without accurate pose initialization. It employs a local-to-global learning framework to optimize overlapping mini-scenes and synchronize them into a global reference frame, handling mirror symmetry ambiguities.

### [Knowledge NeRF](17854-knowledge-nerf.md)
A few-shot novel view synthesis method for dynamic articulated objects that reconstructs a current state using sparse inputs by leveraging a pretrained NeRF model of a previous state as a knowledge base. It uses a lightweight projection module to map points from the current state to the knowledge base, avoiding the limited clarity of monocular video deformation methods.

### [Relighting and Shading](17925-relighting-and-shading.md)
Techniques that address lighting inconsistency when compositing objects into NeRF scenes or synthesizing novel views under varying illumination. Unlike standard methods that treat radiance as static, they explicitly model lighting interactions, inter-reflections, or shading correlations to enable physically plausible relighting and HDR synthesis.

### [View Synthesis and Morphing](17926-view-synthesis-and-morphing.md)
Not given in the sources.

### [Sparsesat-NeRF](17972-sparsesat-nerf.md)
An optimization method for sparse-view satellite imagery that integrates dense depth supervision from Semi-Global Matching to resolve few-view reconstruction ambiguities. It uses a single coarse network with a hybrid ray sampling strategy (random for global structure, guided Gaussian around depth priors for local geometry) and explicitly models input depth uncertainty.

### StegaNeRF
A method for steganographic information embedding in NeRFs that allows accurate hidden information extraction from rendered images while preserving visual quality [S1]. Unlike 2D steganography, it hides information in a continuous neural representation, enabling viewpoint-independent recovery for copyright identification [S1].

### MapNeRF
Synthesizes out-of-trajectory driving views with semantic road consistency by incorporating map priors into neural radiance fields [S1]. It uses coarse ground surface and lane information as uncertain supervisory signals to guide the density field and depth warping, distinguishing itself from siblings that rely solely on image-based augmentation [S1].

### [Radiance Field Decoding with Hybrid Features](18187-radiance-field-decoding-with-hybrid-features.md)
Defines the radiance field decoder that predicts volume density and color for a 3D location and viewing direction, conditioned on global triplane features and residual local features. It replaces raw positional encoding with structured global features and local residuals, enabling effective 360° view synthesis from minimal input views.

### [Depth and Geometry Priors](18223-depth-and-geometry-priors.md)
Optimization techniques that incorporate explicit depth information into NeRF training to resolve scale ambiguity and geometric inaccuracies. They differ from sibling methods by directly supervising geometric output rather than just photometric output, enforcing metric consistency and improving reconstruction speed in sparse-view scenarios.

### [NAN (Noise-Aware NeRFs for Burst-Denoising)](20287-nan-noise-aware-nerfs-for-burst-denoising.md)
A NeRF-based burst denoising method that leverages inter-view and spatial information to handle high noise and large camera motion in 3D space. It treats the burst as an un-ordered set with potentially large parallax, explicitly reasoning about the scene's underlying 3D structure to cope with significant motion and occlusions under very high noise levels.

### [JacobiNeRF Label Propagation](20307-jacobinerf-label-propagation.md)
A test-time method for transferring semantic or instance segmentation labels from a source view to novel target views in a trained NeRF. It perturbs network parameters along gradients of mutual information associated with labeled pixels, synthesizing perturbed images to assign labels, leveraging the second-order geometry of the radiance field.

### UNISURF
A hybrid neural representation that unifies volume rendering and surface rendering to achieve high-quality geometry and sharp textures without mask supervision [S2, S3]. It replaces the density function with an occupancy network, leveraging volume rendering in early optimization stages and surface rendering in later stages to avoid blurry texture mapping [S2, S3].

### [Deformation and Dynamic Modeling](21915-deformation-and-dynamic-modeling.md)
Extends static radiance field representations to handle temporal changes by introducing time as an explicit input and using deformation fields, scene flow, or canonical space mappings. Unlike static variants, these methods explicitly model motion, distinguishing themselves from siblings that handle lighting changes or interpolation without physical motion modeling [G].

### [Hierarchical and Local Scene Decomposition](22843-hierarchical-and-local-scene-decomposition.md)
Addresses memory constraints and convergence issues of global NeRFs on large-scale scenes by partitioning the scene into multiple local radiance fields. Unlike global implicit representations, these methods use structured collections of local fields to enable efficient, structure-aware scene decomposition and interactive editing.

### [Activation-based Density Estimation for NeRF Acceleration](23005-activation-based-density-estimation-for-nerf-acceleration.md)
Accelerates inference in pre-trained NeRFs by deriving density estimates from intermediate layer activations of the coarse MLP. Unlike coarse-to-fine sampling or sampling networks, it uses existing capacity from a single coarse NeRF's intermediate features to reduce computational requirements by up to 50% with minimal quality loss.

### [Event-based and Spiking Neural Network Inputs](23025-event-based-and-spiking-neural-network-inputs.md)
Not given in the sources.

### Geometry Composition by Segmented Sampling
A variant for inserting objects into scenes. It transforms sampling points from scene coordinates to object coordinates using the object's insertion position (translation and rotation) [S3]. It composites density as the normalized sum of densities from the scene and object at the same position, and composites reflectance as a weighted sum of reflectance values weighted by voxel particle densities [S3].

### Neural Proxy Rendering with Mixed Samplers
A rendering pipeline that mixes a neural proxy sampler $S_p$ for animated objects and a neural scene sampler $S_c$ for static elements [S4, unofficial]. It separates particles into a proxy set $X_p$ and scene set $X_c$, finds the convex set bounded by intersections with the proxy mesh using triangle intersection methods to determine Barycentric coordinates, and samples the neural texture locally [S4, unofficial].

### StructNeRF
Enhances new view creation from limited inputs in indoor settings by integrating structural indicators into self-guided depth estimation and applying planar coherence loss [S5]. It limits textured area shapes via patch-based multi-view coherent photometric loss and implements progressive training to reduce noise from limited depth point clouds in structure-from-motion [S5].

### SHERF
A generalizable human NeRF model for recovering animatable 3D humans from a single input image, forecasting varied depth assessments from each perspective [S5]. It uses conditional implicit maximum likelihood estimation (cIMLE) to address intrinsic uncertainty in single-eye depth estimation and presents a spatial sculpting loss technique to merge theoretical depth maps across perspectives [S5].

### IR-NeRF
A NeRF training method without camera poses that improves robustness by introducing implicit pose regularization via a scene codebook [S5]. It implicitly captures scene-specific camera pose distribution as a priori, consistently outperforming GNeRF in image quality metrics (PSNR, SSIM, LPIPS) [S5].

### BOF-NeRF
Incorporates bidirectional optical flow to supervise and guide the network to learn scene structure and synthesize novel views under fewer input views [S6]. It estimates camera parameters using COLMAP, calculates forward/backward optical flow using RAFT, determines effective optical flow areas, and applies view-enhanced fusion to refine novel view details [S6].

### Low-Resolution Feature Upsampling
Approximates high-resolution feature maps by upsampling low-resolution feature maps, allowing volume rendering to only generate low-resolution feature maps [S7, unofficial]. It introduces multi-view inconsistency because pixel-space operations like upsampling compromise 3D consistency and capture spurious correlations in 2D observations [S7, unofficial].

### Hybrid Neural Rendering Model
Combines image-based and neural 3D representations to render high-quality, view-consistent images for large-scale scenes [S8]. It extracts multiscale image features from nearby views, projects query points to retrieve features, constructs image-based features, obtains neural 3D features via k-nearest neighbor interpolation, and aggregates them using a 3D-guided fusion module [S8].

### URF
Exploits additional LIDAR data to supervise depth prediction in large-scale scenarios [S9, unofficial].

### SDF-induced Volume Rendering
Computes surface normal $\mathbf{n}(\mathbf{r})$ at the intersection point as $\mathbf{n}(\mathbf{r}) = \nabla f(\mathbf{o} + \mathbf{d}(\mathbf{r})\mathbf{v})$ [S10].

### Stereo Radiance Fields
Extracts features from several context views and leverages learned correspondence matching between pairwise features across context images to aggregate features [S9, unofficial].

### DeRF
Decomposes the scene into 'soft Voronoi diagrams' to take advantage of accelerator memory architectures [S11, unofficial].

### Ha-NeRF
Extends NeRF in the Wild to hallucinate novel appearances [S9, unofficial].

### PS-NeRF
A fusion-based approach that combines NeRF with photometric stereo methods to improve mesh resolution [S12].

### NeRF Re-animation
Establishes a canonical space using NeRF representation, constructs a neural deformation field constrained by SMPL, feeds the output into a position encoder, and converts the encoder output into shape features and volume density via a density model [S13].

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources. The model uses 8.4M parameters (early-stop) or 16.8M parameters (fully-trained) [S1].
*   **Measured Performance:** On the Blender dataset, Hyb-NeRF (early-stop, 8.4M params) achieves PSNR 33.28 dB and SSIM 0.960, trained in 4 minutes with 40k iterations; fully-trained (16.8M params) achieves PSNR 33.94 dB and SSIM 0.964, trained in 9 minutes with 40k iterations [S1].
*   **Scaling:** Uses Adam optimizer with $\beta_1=0.9, \beta_2=0.99$, learning rate $5\times10^{-3}$, and batch size 256K samples [S1].
*   **Strengths:** Fewer trainable parameters than Instant NGP (8.4M/16.8M vs 12.6M/24.4M) due to larger $N_{\min}$ and smaller $L_f$; fixed positional encoding at coarse levels guarantees convergence while trainable fine-level grids capture high-frequency details [S1].
*   **Limitations:** Shallow MLPs have limited representation power without cone tracing features; multi-resolution grids require careful handling to avoid aliasing artifacts [S1].
*   **When to use:** Suitable for high-quality, fast novel view synthesis on consumer hardware where memory efficiency and rapid convergence are prioritized [S1].
*   **When not:** Unsuitable for scenes requiring explicit surface extraction without auxiliary methods, or where strict metric scale is required without depth priors [G].

## Background
HybridNeRF was introduced in 2024 by the authors of "Hyb-NeRF: A Multiresolution Hybrid Encoding for Neural Radiance Fields" [S1]. It builds on multi-resolution hash encoding and learnable positional encoding concepts to balance rendering quality and computational efficiency [S1]. The method was evaluated on Blender, Synthetic-NSVF, and Tanks&Temples datasets, demonstrating state-of-the-art performance with significantly reduced training times compared to prior methods [S1].

## Sources
[S1] Hyb-NeRF: A Multiresolution Hybrid Encoding for Neural Radiance Fields, https://doi.org/10.1109/wacv57701.2024.00365, fetched 2026-10-26
[S2] HybridNeRF: Efficient Neural Rendering via Adaptive Volumetric Surfaces, https://doi.org/10.1109/cvpr52733.2024.01858, fetched 2026-10-26
[S3] Relighting Scenes With Object Insertions in Neural Radiance Fields, https://doi.org/10.1109/TCSVT.2025.3535599, fetched 2026-10-29
[S4] Neural Proxy: Empowering Neural Volume Rendering for Animation, https://doi.org/10.2312/pg.20211384, fetched 2026-10-26 (unofficial)
[S5] Neural Radiance Field-based Visual Rendering: a Comprehensive Review, https://ieeexplore.ieee.org/abstract/document/11455380, fetched 2026-10-30
[S6] Bidirectional Optical Flow NeRF: High Accuracy and High Quality under Fewer Views, https://doi.org/10.1609/aaai.v37i1.25109, fetched 2026-10-29
[S7] StyleNeRF: A Style-based 3D-Aware Generator for High-resolution Image Synthesis, https://doi.org/10.48550/arxiv.2110.08985, fetched 2026-10-26 (unofficial)
[S8] Hybrid Neural Rendering for Large-Scale Scenes with Motion Blur, http://arxiv.org/abs/2304.12652v1, fetched 2026-10-27
[S9] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
[S10] HelixSurf: A Robust and Efficient Neural Implicit Surface Learning of Indoor Scenes with Iterative Intertwined Regularization, https://arxiv.org/abs/2302.14340v2, fetched 2026-10-26
[S11] Neural Volume Rendering: NeRF And Beyond, https://arxiv.org/abs/2101.05204v2, fetched 2026-10-26 (unofficial)
[S12] A Critical Analysis of NeRF-Based 3D Reconstruction, https://doi.org/10.3390/rs15143585, fetched 2026-10-26
[S13] Human 3D Avatar Modeling with Implicit Neural Representation: A Brief Survey, https://doi.org/10.1109/icsps58776.2022.00148, fetched 2026-10-26
