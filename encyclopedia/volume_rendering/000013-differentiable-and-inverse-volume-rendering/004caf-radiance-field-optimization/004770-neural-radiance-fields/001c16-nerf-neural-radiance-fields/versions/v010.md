# NeRF (Neural Radiance Fields)

Also called: Mildenhall et al. 2020, Neural Radiance Field (NeRF), Original NeRF, Radiance Fields (NeRF)

## Overview
NeRF represents a static 3D scene as a continuous volumetric function that maps a 3D spatial position and a 2D viewing direction to an emitted RGB color and a volume density scalar [S1][S3, unofficial][S5]. It solves the problem of novel view synthesis by replacing explicit geometric primitives (meshes or point clouds) with an implicit neural representation encoded in network weights, allowing the rendering of photorealistic images from arbitrary camera viewpoints not seen during training [S1][S7, unofficial]. It distinguishes itself from earlier implicit methods by explicitly modeling view-dependent appearance (specularities, transparency) alongside geometry, and by using a two-stage hierarchical sampling strategy with positional encoding to overcome the spectral bias of standard MLPs [S2, S6, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ posed RGB images $\{I_i\}$ with known camera poses (extrinsics $R_i, t_i$ and intrinsics $K_i$).
*   **Outputs:** A trained neural network (or pair of networks) representing the radiance field, and rendered novel views.

**Data Structures:**
The core data structure is a Multi-Layer Perceptron (MLP), denoted $F_\Theta$. In the original formulation, the network is split into two branches to enforce multi-view consistency:
1.  **Density Network ($F_{\Theta_1}$):** Takes encoded 3D position $\gamma_x(x)$ as input. Outputs a scalar volume density $\sigma \in \mathbb{R}^+$ and a 256-dimensional intermediate feature vector $z \in \mathbb{R}^{256}$ [S5, S34].
2.  **Color Network ($F_{\Theta_2}$):** Takes the feature vector $z$ and the encoded viewing direction $\gamma_d(d)$ as input. Outputs an RGB color vector $c \in \mathbb{R}^3$ [S5, S34].

**Positional Encoding:**
To enable the MLP to learn high-frequency details, input coordinates are mapped to a higher-dimensional space using sinusoidal positional encoding $\gamma(\cdot)$ [S3, S17, unofficial]. For an input scalar $p$ and frequency level $L$, the encoding is:
$$ \gamma(p) = (\sin(2^0 \pi p), \cos(2^0 \pi p), \dots, \sin(2^{L-1} \pi p), \cos(2^{L-1} \pi p)) $$
This is applied element-wise to the 3D position $x$ and the 3D viewing direction $d$ [S3, S17, unofficial]. The viewing direction is typically represented as a unit vector $d \in \mathbb{R}^3$ derived from spherical angles $(\theta, \phi)$ [S5].

**Numbered Steps:**

1.  **Ray Generation:** For each pixel $(u, v)$ in the target image, compute the ray origin $o$ and direction $d$ in world coordinates using the camera pose and intrinsics [S15, S25, unofficial].
    $$ d_{cam} = K^{-1} \begin{bmatrix} u \\ v \\ 1 \end{bmatrix}, \quad d = R \cdot d_{cam} $$
    $$ o = -R^T t $$

2.  **Sampling:**
    *   **Coarse Sampling:** Divide the ray interval $[t_n, t_f]$ into $N_c$ evenly spaced bins. Draw one sample $t_i$ uniformly from each bin (stratified sampling) to generate $N_c$ points $x_i = o + t_i d$ [S4][S6, unofficial].
    *   **Fine Sampling:** Query the coarse network to get densities $\sigma_i$. Compute weights $w_i = T_i (1 - \exp(-\sigma_i \delta_i))$. Resample $N_f$ points from the distribution proportional to $w_i$ to generate fine points [S6, S21, unofficial].

3.  **Network Query:**
    *   For each sample point $x_i$ and direction $d$:
        *   Compute $z_i = F_{\Theta_1}(\gamma_x(x_i))$.
        *   Compute $\sigma_i$ from the last layer of $F_{\Theta_1}$ (or directly if single network).
        *   Compute $c_i = F_{\Theta_2}(\gamma_d(d), z_i)$ [S5, S34].

4.  **Volume Rendering:**
    Compute the accumulated transmittance $T_i$ and opacity $\alpha_i$ for each sample:
    $$ \delta_i = t_{i+1} - t_i $$
    $$ \alpha_i = 1 - \exp(-\sigma_i \delta_i) $$
    $$ T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right) $$
    The final color $\hat{C}(r)$ is the weighted sum:
    $$ \hat{C}(r) = \sum_{i=1}^{N} T_i \alpha_i c_i $$
    [S9][S21, unofficial]

5.  **Optimization:**
    Minimize the Mean Squared Error (MSE) between the rendered color $\hat{C}(r)$ and the ground truth pixel color $C(r)$ over all training rays [S3, S6, unofficial]:
    $$ \mathcal{L} = \frac{1}{n} \sum_{i=1}^n \| I_i - \hat{I}_i(\sigma, c) \|_2^2 $$
    Parameters $\Theta$ are updated via stochastic gradient descent [S6, unofficial].

**Parameters:**
*   $N_c$: Number of coarse samples (typically 64) [S28, unofficial].
*   $N_f$: Number of fine samples (typically 128) [S28, unofficial].
*   $L$: Number of frequency bands in positional encoding (typically 10) [S3, S17, unofficial].
*   Hidden layer size: 256 neurons [S5, S34].
*   Number of hidden layers: 8 for density, 1 for color (in the split architecture) [S34].

**Edge Cases and Pitfalls:**
*   **Spectral Bias:** Standard MLPs struggle with high-frequency signals. Positional encoding is required to capture fine details [S3, S39, unofficial].
*   **View-Dependent Artifacts:** If the viewing direction is injected too early in the network, it can cause view-dependent artifacts in the density field. The original NeRF injects direction only into the color head to ensure density is view-independent [S3, S21, unofficial].
*   **Unbounded Scenes:** Standard bounding boxes $[t_n, t_f]$ fail for unbounded scenes. Variants like NeRF++ use inverted sphere parameterization [S3, unofficial].

## Subtopics

### [NeRF Training Configuration](1203-nerf-training-configuration.md)
Focuses on hyperparameters and sparsity priors to condense volume density for sparse representations.

### [NeRF-SH](1348-nerf-sh.md)
Replaces RGB output with spherical harmonic coefficients to enable sparse voxel storage (PlenOctree).

### [Recursive-NeRF](3029-recursive-nerf.md)
Uses a hierarchy of branch networks based on uncertainty to adaptively refine complex regions.

### [Mip-NeRF](3381-mip-nerf.md)
Renders anti-aliased conical frustums instead of rays to handle multi-scale details.

### [NeMF (Neural Microflake Field)](3387-nemf-neural-microflake-field.md)
Models light transportation via microflake normals for translucent objects.

### [EgoNeRF](4096-egonerf.md)
Uses spherical feature grids for efficient large-scale egocentric capture.

### [Intrinsic NeRF](4623-intrinsic-nerf.md)
Decouples geometry, illumination, and material properties for relighting.

### [Sparse Neural Volume Representation](7055-sparse-neural-volume-representation.md)
Uses sparse 3D grids of per-voxel features for scalability.

### [Instant Neural Graphics Primitives (Instant NGP)](8637-instant-neural-graphics-primitives-instant-ngp.md)
Uses multi-resolution hash encoding for fast training and rendering.

### [Neural Radiance Distribution Field](10216-neural-radiance-distribution-field.md)
Maps rays directly to parametric distributions for real-time rendering.

### [Incremental CONfidence (ICON)](11446-incremental-confidence-icon.md)
Eliminates initial SfM by incrementally registering streaming video.

### [Bayesian Uncertainty Quantification for SeaThru-NeRF](11861-bayesian-uncertainty-quantification-for-seathru-nerf.md)
Quantifies spatial uncertainty for underwater reconstruction.

### [Control-NeRF](13260-control-nerf.md)
Decouples scene-specific features from a shared network for 3D-aware editing.

## Operating characteristics
*   **Time Complexity:** Training is slow, typically requiring 10-20 hours on a single GPU for a single scene [S1, S5]. Rendering is also slow, taking ~30 seconds per image at 1080p due to the need to query the MLP hundreds of times per ray [S1, S5].
*   **Memory Complexity:** The model size is compact (~5-10 MB for the MLP weights) [S11, S28, unofficial], but training requires storing gradients for all parameters, and rendering requires storing intermediate transmittance values.
*   **Performance:** Achieves state-of-the-art photorealism on bounded scenes (e.g., Blender synthetic data) but struggles with unbounded scenes and reflective surfaces without modifications [S1, S4].
*   **Strengths:** High visual fidelity, handles view-dependent effects (specularity) naturally, compact representation.
*   **Limitations:** Slow training and rendering, poor surface extraction (density is entangled with appearance), sensitive to camera pose errors [S1, S5].

## Background
NeRF was introduced by Mildenhall et al. in 2020 [S1, S5]. It builds on the concept of Implicit Neural Representations (INRs) and differentiable volume rendering. The method was motivated by the limitations of explicit 3D representations and the need for high-quality novel view synthesis from sparse inputs. It established the field of neural rendering, inspiring numerous variants and accelerations [S1, S5].

## Sources
[S1] Neural Rendering Techniques, https://doi.org/10.5281/zenodo.19614729, fetched 2026-10-26
[S2] Fourier Feature Networks and Neural Volume Rendering, https://github.com/matajoh/fourier_feature_nets, fetched 2026-10-26 (unofficial)
[S3] NeRF++: Analyzing and Improving Neural Radiance Fields, https://doi.org/10.48550/arxiv.2010.07492, fetched 2026-10-26 (unofficial)
[S4] Specularity in NeRFs: A Comparative Study of Ref-NeRF and NRFF, https://doi.org/10.5201/ipol.2025.562, fetched 2026-10-26
[S5] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S6] From reconstruction to reasoning : neural radiance fields for comprehensive 3D scene understanding, https://theses.hal.science/tel-05693528v1, fetched 2026-10-26 (unofficial)
[S7] NeRF Explained Simply | The 5D Function & Volumetric Rendering, https://www.youtube.com/watch?v=BL-KcFm4Wms, fetched 2026-10-27 (unofficial)
[S8] DP-NeRF: Deblurred Neural Radiance Field with Physical Scene Priors, https://arxiv.org/abs/2211.12046v4, fetched 2026-10-26
[S9] MS-NeRF: Multi-Space Neural Radiance Fields, https://doi.org/10.1109/tpami.2025.3540074, fetched 2026-10-26
[S10] Review of Key Technologies and Application Scenarios of Neural Rendering, https://doi.org/10.54254/2755-2721/2026.36116, fetched 2026-10-26
[S11] yenchenlin/nerf-pytorch, https://github.com/yenchenlin/nerf-pytorch, fetched 2026-10-26 (unofficial)
[S12] Radar Fields: An Extension of Radiance Fields to SAR, https://doi.org/10.1109/cvprw63382.2024.00061, fetched 2026-10-26
[S13] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S14] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S15] yashtmy/nerf-3d-vision, https://github.com/yashtmy/nerf-3d-vision, fetched 2026-10-26 (unofficial)
[S16] PDF Volume Rendering - faculty.cc.gatech.edu, https://faculty.cc.gatech.edu/~hays/compvision2025spring/lectures/26.pdf, fetched 2026-10-26 (unofficial)
[S17] GRAF: Generative Radiance Fields for 3D-Aware Image Synthesis, https://doi.org/10.48550/arxiv.2007.02442, fetched 2026-10-26 (unofficial)
[S18] SPARF: Neural Radiance Fields from Sparse and Noisy Poses, https://doi.org/10.1109/cvpr52729.2023.00408, fetched 2026-10-26
[S19] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
[S20] High-fidelity 3D Reconstruction of Plants using Neural Radiance Field, https://arxiv.org/abs/2311.04154v1, fetched 2026-10-27
[S21] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S22] NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis | Takuma Yoneda, https://takuma.yoneda.xyz/posts/nerf, fetched 2026-10-27 (unofficial)
[S23] Understanding NeRFs - by Cameron R. Wolfe, Ph.D., https://cameronrwolfe.substack.com/p/understanding-nerfs, fetched 2026-10-27 (unofficial)
[S24] charan2598/NeRF-PyTorch, https://github.com/charan2598/NeRF-PyTorch, fetched 2026-10-27 (unofficial)
[S25] sr-bang/NeRF, https://github.com/sr-bang/NeRF, fetched 2026-10-27 (unofficial)
[S26] Neural Radiance Fields - Lei Mao's Log Book, https://leimao.github.io/blog/Neural-Radiance-Fields-NeRF/, fetched 2026-10-27 (unofficial)
[S27] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
[S28] Neural radiance fields (NeRF), https://www.ultralytics.com/glossary/neural-radiance-fields-nerf, fetched 2026-10-28 (unofficial)
[S29] Generalizing Neural Radiance Fields - BAIR Commons, https://bcommons.berkeley.edu/generalizing-neural-radiance-fields, fetched 2026-10-28 (unofficial)
[S30] Taichi NeRF (Part 1): Develop and Deploy Instant NGP without writing CUDA | Taichi Docs, https://docs.taichi-lang.org/blog/taichi-instant-ngp, fetched 2026-10-29 (unofficial)
[S31] A Visual Exploration of Neural Radiance Fields, https://djrhails.github.io/post--radiance-fields/, fetched 2026-10-29 (unofficial)
[S32] FlipNeRF: Flipped Reflection Rays for Few-shot Novel View Synthesis, https://doi.org/10.1109/iccv51070.2023.02092, fetched 2026-10-29
[S33] HyperNeRFGAN: Hypernetwork approach to 3D NeRF GAN, https://doi.org/10.48550/arxiv.2301.11631, fetched 2026-10-29 (unofficial)
[S34] Neural Radiance Fields in der Fabrikplanung/On the applicability of Neural Radiance Fields for virtual model reconstruction in factory planning, https://doi.org/10.37544/1436-4980-2023-06-11, fetched 2026-10-29
[S35] Methods and Strategies for Improving the Novel View Synthesis Quality of Neural Radiation Field, https://doi.org/10.1109/access.2024.3382997, fetched 2026-10-30
[S36] Reference-guided Controllable Inpainting of Neural Radiance Fields, https://doi.org/10.1109/ICCV51070.2023.01633, fetched 2026-10-29
[S37] DoubleField: Bridging the Neural Surface and Radiance Fields for High-fidelity Human Reconstruction and Rendering, https://doi.org/10.1109/cvpr52688.2022.01541, fetched 2026-10-29
[S38] Vision through Obstacles—3D Geometric Reconstruction and Evaluation of Neural Radiance Fields (NeRFs), https://doi.org/10.3390/rs16071188, fetched 2026-10-30
[S39] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
[S40] ニューラル放射輝度場 NeRF 入門, https://zenn.dev/abist_inno/articles/353f36119d6b39, fetched 2026-10-30 (unofficial)
[S41] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
