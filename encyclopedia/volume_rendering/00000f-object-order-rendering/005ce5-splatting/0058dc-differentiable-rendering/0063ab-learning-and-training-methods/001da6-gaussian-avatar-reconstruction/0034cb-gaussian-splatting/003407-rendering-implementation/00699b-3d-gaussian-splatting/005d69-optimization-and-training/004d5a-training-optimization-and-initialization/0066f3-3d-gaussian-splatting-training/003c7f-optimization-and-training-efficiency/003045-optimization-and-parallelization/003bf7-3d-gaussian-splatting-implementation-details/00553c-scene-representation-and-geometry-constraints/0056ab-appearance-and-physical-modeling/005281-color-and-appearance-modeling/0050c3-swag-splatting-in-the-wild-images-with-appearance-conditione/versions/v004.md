# SWAG (Splatting in the Wild images with Appearance-conditioned Gaussians)

## Overview
SWAG is an extension of 3D Gaussian Splatting (3DGS) designed to reconstruct 3D scenes from unstructured, in-the-wild image collections that exhibit significant photometric variations and contain transient occluders (e.g., tourists, clouds) [S1]. It distinguishes itself from standard 3DGS and its sibling [3iGS](13977-3igs-factorised-tensorial-illumination-for-3d-gaussian-splat.md) by explicitly modeling appearance variations via per-image embeddings and disentangling static scene geometry from transient objects through adaptive opacity modulation [S1]. While 3DGS averages appearance across views leading to color shifts and floaters in unstructured data, SWAG learns a latent appearance space and isolates transient Gaussians to prevent them from corrupting the static reconstruction [S1].

## How it works
SWAG augments the standard 3DGS Gaussian attributes with a conditional appearance model and a transient detection mechanism. The method optimizes a set of Gaussians where each Gaussian $i$ retains standard attributes: centroid $\mathbf{x}_i$, scale $S_i$, rotation $R_i$, base opacity $\alpha_i$, and base color $\mathbf{c}_i$ (encoded in spherical harmonic coefficients) [S1]. Additionally, it optimizes three new components: a Hash Grid encoding for the Gaussian centers $\text{emb}(\mathbf{x}_i)$, a per-image embedding vector $\mathbf{l}_I$ for each training image $I$, and a Multi-Layer Perceptron (MLP) $F_\theta$ [S1].

### Data Structures and Layout
1.  **Gaussian Attributes**: For each Gaussian $i$:
    *   $\mathbf{x}_i \in \mathbb{R}^3$: Centroid.
    *   $S_i \in \mathbb{R}^3$: Scale (log-space optimization).
    *   $R_i \in \mathbb{R}^4$: Rotation (quaternion).
    *   $\alpha_i \in \mathbb{R}$: Base opacity.
    *   $\mathbf{c}_i$: Base color (SH coefficients).
2.  **Hash Grid Encoding**: A hash table of size $T = 2^{19}$ maps 3D positions to feature vectors. It uses $L=12$ levels, with finest resolution $N_{\max}=2048$ and coarsest resolution $N_{\min}=16$ [S1]. The output is $\text{emb}(\mathbf{x}_i)$.
3.  **Image Embeddings**: A learnable vector $\mathbf{l}_I \in \mathbb{R}^{24}$ for each training image $I$ [S1].
4.  **MLP $F_\theta$**: An 8-layer network with input dimension 3 (concatenated features), hidden layer size 51, and output dimension 64 [S1].

### Rendering Equation
The final color $\hat{\mathbf{C}}$ for a pixel is computed via standard alpha blending [S1]:
$$ \hat{\mathbf{C}} = \sum_{i \in N} T_i g_i(\mathbf{u} | \boldsymbol{\mu}', \boldsymbol{\Sigma}') \alpha_i \mathbf{c}_i $$
where $T_i$ is the accumulated transmittance, $g_i$ is the projected Gaussian density, and $\mathbf{c}_i$ is the final color predicted by the appearance model [S1].

### Appearance and Transient Modeling
For a Gaussian $i$ rendered from image $I$, the appearance model $F_\theta$ takes three inputs:
1.  The Gaussian's base color $\mathbf{c}_i$.
2.  The associated image embedding $\mathbf{l}_I$.
3.  The encoded center $\text{emb}(\mathbf{x}_i)$ from the Hash Grid [S1].

The MLP $F_\theta$ outputs two values:
1.  An image-dependent color correction $\Delta \mathbf{c}_I$.
2.  An image-dependent opacity variation parameter $\Delta \alpha_I$ [S1].

The final color $\mathbf{c}_i$ used in the rendering equation is derived from $\mathbf{c}_i$ and $\Delta \mathbf{c}_I$ (implementation detail: typically additive or multiplicative modulation, though the source specifies the output is $\mathbf{c}_I$ and $\Delta \alpha_I$; we assume $\mathbf{c}_i$ is modulated to produce the final radiance contribution). The opacity variation $\Delta \alpha_I$ is treated as the location variable of a concrete distribution (Gumbel-Softmax or similar reparameterization trick) to allow differentiable sampling of the opacity variation $\widetilde{\Delta \alpha}_I$ [S1]. The effective opacity for the Gaussian in image $I$ is modulated by this sampled variation [S1].

### Optimization Steps
1.  **Initialization**: Initialize Gaussians using SfM points from the unstructured image collection [S1].
2.  **Forward Pass**:
    *   For each training image $I$, render the scene.
    *   For each visible Gaussian $i$, compute $\text{emb}(\mathbf{x}_i)$ using the Hash Grid.
    *   Concatenate $\mathbf{c}_i$, $\mathbf{l}_I$, and $\text{emb}(\mathbf{x}_i)$.
    *   Pass through $F_\theta$ to get $\Delta \mathbf{c}_I$ and $\Delta \alpha_I$.
    *   Sample $\widetilde{\Delta \alpha}_I$ from the concrete distribution defined by $\Delta \alpha_I$.
    *   Compute final color and effective opacity.
3.  **Loss Calculation**: Minimize a combination of L1 loss and D-SSIM loss between the rendered image $\hat{\mathbf{C}}$ and the ground truth $\mathbf{C}_{gt}$ [S1].
4.  **Backpropagation**: Update Gaussian attributes, Hash Grid features, image embeddings $\mathbf{l}_I$, and MLP weights $\theta$ [S1].
5.  **Densification**: Apply 3DGS densification (splitting/pruning) every 2000 iterations until iteration 30,000 [S1].

### Parameters and Hyperparameters
*   **Iterations**: 60,000 [S1].
*   **Densification Interval**: 2,000 [S1].
*   **Densification End**: 30,000 [S1].
*   **Image Embedding Dimension**: 24 [S1].
*   **Learning Rate (Image Embeddings)**: 0.0025 [S1].
*   **Hash Grid Size ($T$)**: $2^{19}$ [S1].
*   **Hash Grid Resolutions**: $N_{\max}=2048$, $N_{\min}=16$ [S1].
*   **Hash Grid Levels ($L$)**: 12 [S1].
*   **Learning Rate (Hash Grid)**: 0.0025 [S1].
*   **MLP Layers**: 8 [S1].
*   **MLP Input Dimension**: 3 (concatenated features) [S1].
*   **MLP Hidden Size**: 51 [S1].
*   **MLP Output Dimension**: 64 [S1].
*   **MLP Learning Rate**: 0.0025 [S1].

### Edge Cases and Pitfalls
*   **Transient Detection Threshold**: Transient Gaussians are identified by low variance in their sampled opacity variations ($\text{Var}(\widetilde{\Delta \alpha}_I)$) across images [S1]. However, frequent transient regions (e.g., clouds, crowds) may be incorrectly classified as transient, leading to holes (black spots) if all such Gaussians are removed [S1].
*   **Hole Mitigation**: To avoid holes, Gaussians can be removed based on an adaptive threshold $\lambda$ rather than requiring zero variance ($\text{Var}(\widetilde{\Delta \alpha}_I) < \lambda$), but this requires manual tuning of $\lambda$ [S1].
*   **Performance**: Training time almost doubles compared to 3DGS, and inference time per frame is 10 times longer [S1].

## Operating characteristics
*   **Time Complexity**: Training time is approximately 2x that of 3DGS due to the MLP and Hash Grid computations [S1]. Inference time per frame is 10x slower than 3DGS [S1].
*   **Memory Complexity**: Increased memory usage due to the Hash Grid ($2^{19}$ entries), image embeddings (24 dimensions per image), and MLP weights [S1].
*   **Strengths**: Handles unstructured in-the-wild data with lighting variations and transient occluders without supervision. Disentangles static and dynamic Gaussians, preventing blurry elongated Gaussians common in view-dependent 3DGS [S1]. Enables controllable appearance via interpolation of image embeddings [S1].
*   **Limitations**: Sensitive to the threshold $\lambda$ for transient removal; improper tuning leads to holes. Slower training and inference than standard 3DGS.
*   **When to Use**: Reconstruction of outdoor landmarks or tourist sites from internet photo collections where lighting changes and transient objects are present [S1].
*   **When Not to Use**: Structured studio datasets with controlled lighting and no transient objects, where standard 3DGS is faster and sufficient [S1].

## Background
SWAG was introduced by H. Dahmani et al. in 2024 [S1]. It addresses the limitations of 3D Gaussian Splatting when applied to unstructured in-the-wild image collections, which often suffer from poor performance due to photometric variations and transient occluders [S1]. The method builds on the principles of differentiable volume rendering and 3DGS, extending them with appearance conditioning and transient detection mechanisms [S1]. It has been evaluated on the Phototourism Dataset (Brandenburg Gate, Sacré Cœur, Trevi Fountain) and the NeRF-OSR Dataset [S1].

## Sources
[S1] SWAG: Splatting in the Wild images with Appearance-conditioned Gaussians, https://doi.org/10.48550/arXiv.2403.10427, fetched 2026-10-29
