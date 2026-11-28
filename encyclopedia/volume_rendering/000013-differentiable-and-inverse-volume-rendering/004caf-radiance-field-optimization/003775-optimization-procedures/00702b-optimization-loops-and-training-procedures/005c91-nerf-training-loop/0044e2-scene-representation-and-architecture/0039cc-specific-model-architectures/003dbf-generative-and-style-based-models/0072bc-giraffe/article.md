# GIRAFFE

Also called: Compositional Generative Neural Feature Fields, GIRAFFE Object Representation, GIRAFFE Scene Composition, Generative Inverse Radiance Fields, Generative Neural Feature Fields

## Overview
GIRAFFE represents scenes as compositions of $N$ entities (typically $N-1$ foreground objects and 1 background), where each entity possesses a separate generative neural feature field and an affine transformation [S1]. This architecture allows for the disentanglement of individual objects' shapes, appearances, and poses from the background without explicit supervision [S4]. It extends the generative NeRF paradigm by enabling the synthesis of complex multi-object scenes through a composition operator that aggregates densities and features from multiple latent-coded NeRFs [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   Camera pose $\xi$ (extrinsics).
*   Number of entities $N$ (sampled from distribution $p_N$ or fixed).
*   For each entity $i \in \{1, \dots, N\}$:
    *   Shape latent code $\mathbf{z}_s^i \sim \mathcal{N}(0, I)$.
    *   Appearance latent code $\mathbf{z}_a^i \sim \mathcal{N}(0, I)$.
    *   Transformation parameters $\mathbf{T}_i = \{\mathbf{s}_i, \mathbf{t}_i, \mathbf{R}_i\}$ (scale, translation, rotation), sampled from $p_T$.
*   3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$.

**Outputs:**
*   Synthesized RGB image $\hat{\mathbf{I}} \in \mathbb{R}^{H \times W \times 3}$.
*   (Implicitly) Feature image $\mathbf{I}_V \in \mathbb{R}^{H_V \times W_V \times M_f}$.

### Data Structures and Network Layout

**1. Feature Field MLP ($h_\theta$):**
All object feature fields share weights. The background field $h_\theta^N$ uses half the layers and hidden dimension.
*   **Input:** Concatenated positional encoding of transformed position $\gamma(k^{-1}(\mathbf{x}))$, directional encoding $\gamma(k^{-1}(\mathbf{d}))$, shape latent $\mathbf{z}_s$, appearance latent $\mathbf{z}_a$.
*   **Positional Encoding:**
    *   Position: $L_x = 2 \cdot 3 \cdot 10 = 60$ dimensions.
    *   Direction: $L_d = 2 \cdot 3 \cdot 4 = 24$ dimensions.
*   **Architecture:**
    *   8 hidden layers (objects) / 4 hidden layers (background).
    *   Hidden dimension: 128 (objects) / 64 (background).
    *   Activation: ReLU.
    *   Skip connections: From input to every 4th layer [G] (standard NeRF architecture implied by "8 layers... ReLU" and standard practice, though specific skip connection details are inherited from the parent [G]).
*   **Output Heads:**
    *   Density head: 1 unit ($\sigma$).
    *   Feature head: $M_f$ units ($\mathbf{f}$). $M_f = 128$ for 64x64 output, $M_f = 256$ for 256x256 output.

**2. Transformation Parameters ($\mathbf{T}_i$):**
*   Scale $\mathbf{s}_i \in \mathbb{R}^3$.
*   Translation $\mathbf{t}_i \in \mathbb{R}^3$.
*   Rotation $\mathbf{R}_i \in SO(3)$.
*   **Background Constraint:** For the background entity ($i=N$), $\mathbf{s}_N$ and $\mathbf{t}_N$ are fixed to span the entire scene and be centered at the origin. $\mathbf{R}_N$ is typically identity or fixed.

**3. 2D Neural Renderer ($\pi_\theta^{\text{neural}}$):**
*   **Input:** Feature image $\mathbf{I}_V$ of resolution $H_V \times W_V$ (typically 16x16).
*   **Architecture:** 2D CNN with Leaky ReLU.
*   **Upsampling:** Combines nearest neighbor upsampling with 3x3 convolutions.
*   **Skip Connections:** Maps feature image to RGB at every spatial resolution via 3x3 conv, adds to previous output via bilinear upsampling.
*   **Output:** Final RGB image $\hat{\mathbf{I}}$ of resolution $H \times W$ (64x64 or 256x256). Sigmoid activation on final layer.

### Numbered Steps

1.  **Sample Latents and Transforms:** For a given scene, sample $N \sim p_N$. For each entity $i$, sample $\mathbf{z}_s^i, \mathbf{z}_a^i \sim \mathcal{N}(0, I)$ and $\mathbf{T}_i \sim p_T$.
2.  **Ray Sampling:** For each pixel in the target image, generate a ray $\mathbf{d}$ from camera pose $\xi$. Sample $N_s = 64$ points $\mathbf{x}_j$ along the ray.
3.  **Transform to Canonical Space:** For each sample point $\mathbf{x}_j$ and direction $\mathbf{d}$, compute the inverse transformation $k_i^{-1}$ for each entity $i$ to map to canonical object space.
    $$ k_i(\mathbf{x}) = \mathbf{R}_i \begin{bmatrix} s_{i,1} & 0 & 0 & 0 \\ 0 & s_{i,2} & 0 & 0 \\ 0 & 0 & s_{i,3} & 0 \\ 0 & 0 & 0 & s_{i,3} \end{bmatrix} \mathbf{x} + \mathbf{t}_i $$
    $$ k_i^{-1}(\mathbf{x}) = \mathbf{R}_i^{-1} \left( \begin{bmatrix} 1/s_{i,1} & 0 & 0 & 0 \\ 0 & 1/s_{i,2} & 0 & 0 \\ 0 & 0 & 1/s_{i,3} & 0 \\ 0 & 0 & 0 & 1/s_{i,3} \end{bmatrix} (\mathbf{x} - \mathbf{t}_i) \right) $$
    [Note: The source matrix for $k(\mathbf{x})$ has a typo in the last row/column index in the text snippet, but standard affine transformation implies the last row is $[0,0,0,1]$ for homogeneous coordinates. The inverse operation applies the inverse scale and rotation.]
4.  **Evaluate Feature Fields:** For each entity $i$, input $\gamma(k_i^{-1}(\mathbf{x}_j))$, $\gamma(k_i^{-1}(\mathbf{d}))$, $\mathbf{z}_s^i$, $\mathbf{z}_a^i$ into $h_\theta$ to obtain density $\sigma_{i,j}$ and feature vector $\mathbf{f}_{i,j}$.
5.  **Compose Entities (Composition Operator):** Combine the outputs from all $N$ entities at point $\mathbf{x}_j$ using the density-weighted mean:
    $$ \sigma_j = \sum_{i=1}^N \sigma_{i,j} $$
    $$ \mathbf{f}_j = \frac{1}{\sigma_j} \sum_{i=1}^N \sigma_{i,j} \mathbf{f}_{i,j} \quad \text{if } \sigma_j > 0, \text{ else } \mathbf{0} $$
6.  **Volume Render Features:** Compute the final feature vector $\mathbf{f}$ for the pixel by integrating along the ray:
    $$ \alpha_j = 1 - \exp(-\sigma_j \delta_j) $$
    $$ \tau_j = \prod_{k=1}^{j-1} (1 - \alpha_k) $$
    $$ \mathbf{f} = \sum_{j=1}^{N_s} \tau_j \alpha_j \mathbf{f}_j $$
    where $\delta_j = \|\mathbf{x}_{j+1} - \mathbf{x}_j\|_2$.
7.  **2D Neural Rendering:** Upsample the low-resolution feature image $\mathbf{I}_V$ (containing $\mathbf{f}$ for all pixels) to high-resolution RGB using $\pi_\theta^{\text{neural}}$.

### Formulas

**Transformation:**
$$ k(\mathbf{x}) = \mathbf{R} \cdot \text{diag}(s_1, s_2, s_3, s_3) \cdot \mathbf{x} + \mathbf{t} $$
[S1]

**Composition:**
$$ C(\mathbf{x}, \mathbf{d}) = \left( \sigma, \frac{1}{\sigma} \sum_{i=1}^N \sigma_i \mathbf{f}_i \right), \quad \sigma = \sum_{i=1}^N \sigma_i $$
[S1]

**Feature Rendering:**
$$ \mathbf{f} = \sum_{j=1}^{N_s} \tau_j \alpha_j \mathbf{f}_j, \quad \tau_j = \prod_{k=1}^{j-1} (1 - \alpha_k), \quad \alpha_j = 1 - e^{-\sigma_j \delta_j} $$
[S1]

**GAN Objective:**
$$ \mathcal{V}(\theta, \phi) = \mathbb{E}[f(D_\phi(G_\theta(\{\mathbf{z}_s^i, \mathbf{z}_a^i, \mathbf{T}_i\}_i, \xi)))] + \mathbb{E}[f(-D_\phi(\mathbf{I})) - \lambda \|\nabla_\phi(\mathbf{I})\|^2] $$
where $f(t) = -\log(1 + \exp(-t))$ and $\lambda = 10$.
[S1]

### Parameters and Typical Values

*   **$N_s$ (Samples per ray):** 64 [S1].
*   **Feature Image Resolution ($H_V \times W_V$):** 16x16 [S1].
*   **Output Resolution ($H \times W$):** 64x64 or 256x256 [S1].
*   **Feature Dimension ($M_f$):** 128 (for 64x64), 256 (for 256x256) [S1].
*   **Generator MLP Layers:** 8 (objects), 4 (background) [S1].
*   **Hidden Dimension:** 128 (objects), 64 (background) [S1].
*   **Positional Encoding Bands:** $L_x = 60$, $L_d = 24$ [S1].
*   **Optimizer:** RMSprop [S1].
*   **Learning Rates:** Generator $1 \times 10^{-4}$ (or $2.5 \times 10^{-4}$ for 256x256), Discriminator $5 \times 10^{-4}$ [S1].
*   **Batch Size:** 32 [S1].
*   **EMA Decay:** 0.999 [S1].
*   **Latent Distribution:** $\mathcal{N}(0, I)$ [S1].
*   **Camera Pose Distribution ($p_\xi$):** Uniform over dataset-dependent elevation angles [S1].
*   **Transformation Distribution ($p_T$):** Uniform over valid object transformations [S1].

### Edge Cases and Pitfalls

*   **Zero Density:** If $\sigma_j = 0$ for all entities at a point, the feature vector $\mathbf{f}_j$ is undefined in the composition operator. The implementation must handle this, typically by setting $\mathbf{f}_j = \mathbf{0}$ [S1].
*   **Computational Cost:** Training and rendering at high resolutions (e.g., 256x256) requires considerable computational power, although it uses less memory than voxel-based techniques [S4].
*   **Background Handling:** The background entity must be fixed to span the scene. Incorrectly allowing background transformation parameters to vary can lead to artifacts or incorrect scene composition [S1].

## Operating characteristics

*   **Time Complexity:** Rendering is dominated by the MLP evaluations for $N_s \times N$ entities per pixel. The 2D neural renderer adds overhead proportional to the upsampling steps.
*   **Memory Complexity:** Lower than voxel-based techniques [S4]. Memory is dominated by the MLP weights and the latent codes. The feature image is small (16x16), reducing VRAM usage during the volume rendering step.
*   **Strengths:** Disentangles object shape, appearance, and pose without explicit supervision [S4]. Allows for compositional scene generation (adding/removing objects) [S1].
*   **Limitations:** High computational cost for training and high-resolution rendering [S4]. Limited to scenes with a manageable number of objects ($N$). Performance depends on the quality of the learned transformation distributions [S1].
*   **When to Use:** When generative control over multiple distinct objects in a scene is required, and explicit 3D annotations are unavailable.
*   **When Not to Use:** For real-time rendering of complex scenes with many objects, or when high-fidelity texture is required without the 2D upsampling step (though GIRAFFE HD addresses this).

## Background

GIRAFFE was introduced by Nagano et al. in 2021 [S1]. It builds upon GRAF [S4] by extending the generative NeRF framework from single-object synthesis to compositional scenes. The method addresses the limitation of previous generative models that could only synthesize simple, single-object scenes [S4]. It is used in research for unsupervised scene decomposition, generative modeling of multi-object datasets (e.g., Chairs, Cats, CelebA), and as a foundation for higher-resolution variants like GIRAFFE HD and UrbanGIRAFFE [S4].

## Sources

[S1] GIRAFFE: Representing Scenes as Compositional Generative Neural Feature Fields, https://github.com/chaerinkong/paper-reviews/issues/63, fetched 2026-10-28
[S2] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
[S3] Neural Rendering and Its Hardware Acceleration: A Review, https://doi.org/10.48550/arxiv.2402.00028, fetched 2026-10-26 (unofficial)
[S4] Semantically-aware Neural Radiance Fields for Visual Scene Understanding: A Comprehensive Review, https://link.springer.com/article/10.1007/s11263-025-02663-5, fetched 2026-10-27
