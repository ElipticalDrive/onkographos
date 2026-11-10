# Mip-NeRF 360


## Overview
Mip-NeRF 360 extends the quality of Neural Radiance Fields (NeRF) to unbounded scenes where the camera may rotate 360 degrees and content exists at arbitrary distances [S2]. It solves the problems of aliasing from multi-scale details and the ambiguity of infinite space by combining anti-aliased cone-casting with a non-linear scene parameterization that maps infinite space into a bounded representation [S1, S2]. It sets itself apart from its parent [Mip-NeRF](link-to-mip-nerf) by introducing a proposal network trained via online distillation and a distortion-based regularizer, allowing it to handle scenes with high dynamic range in depth and scale [S2].

## How it works
Mip-NeRF 360 uses two coupled Multi-Layer Perceptrons (MLPs): a **proposal MLP** ($F_p$) and a **NeRF MLP** ($F_n$) [S2]. The proposal MLP predicts only volumetric density to guide sampling, while the NeRF MLP predicts density and color [S2, S3].

### Inputs and Outputs
*   **Input:** A ray $\mathbf{r}$ defined by origin $\mathbf{o}$ and direction $\mathbf{d}$, and a set of intervals $\mathbf{t}$ along the ray.
*   **Output:** A rendered pixel color $\mathbf{C}(\mathbf{r})$ and gradients for training the MLP weights $\Theta_p$ and $\Theta_n$ [S2].

### Data Structures and Layout
1.  **Proposal MLP ($F_p$):**
    *   Input: Encoded ray intervals.
    *   Output: Scalar density $\tau_p$.
    *   Capacity: Significantly smaller than the NeRF MLP [S2].
2.  **NeRF MLP ($F_n$):**
    *   Input: Encoded ray intervals and viewing direction $\mathbf{d}$.
    *   Output: Density $\tau_n$ and color $\mathbf{c}$ [S2].
    *   Capacity: $\sim 15\times$ larger than the original Mip-NeRF [S2].
3.  **Ray Intervals:**
    *   Represented as conical frustums characterized by mean $\boldsymbol{\mu}$ and covariance $\boldsymbol{\Sigma}$ [S2].
    *   $\boldsymbol{\mu}$ is the center of the frustum along the ray.
    *   $\boldsymbol{\Sigma}$ is the covariance matrix, where the diagonal elements determine the radius of the cone at the frustum's extent, dependent on the camera's focal length and pixel size [S2].

### Scene Parameterization (Contract Mapping)
To handle unbounded scenes, 3D coordinates are mapped to a bounded domain using a contract mapping [S2]. This non-linear parameterization allocates more MLP capacity to nearby content and less to distant content, preventing the "background collapse" and ambiguity inherent in unbounded space [S2]. The specific mapping function is not detailed in the provided sources, but it transforms the input position $\mathbf{x}$ before encoding [S2].

### Integrated Positional Encoding
Standard positional encoding is replaced by **integrated positional encoding** to account for the spatial extent of the conical frustum [S2]. For an interval with mean $\boldsymbol{\mu}$ and covariance $\boldsymbol{\Sigma}$, the encoding $\gamma(\boldsymbol{\mu}, \boldsymbol{\Sigma})$ is:

$$
\gamma(\boldsymbol{\mu}, \boldsymbol{\Sigma}) = \left\{ \begin{bmatrix} \left[ \sin(2^\ell \boldsymbol{\mu}) \exp\left(-2^{2\ell-1} \operatorname{diag}(\boldsymbol{\Sigma})\right) \right] \\ \left[ \cos(2^\ell \boldsymbol{\mu}) \exp\left(-2^{2\ell-1} \operatorname{diag}(\boldsymbol{\Sigma})\right) \right] \end{bmatrix} \right\}_{\ell=0}^{L-1}
$$

where $L$ is the number of frequency bands [S2]. This formula represents the expectation of the standard NeRF encoding with respect to a Gaussian approximation of the conical frustum [S2].

### Training Procedure
The training involves two passes: a coarse pass using the proposal MLP and a fine pass using the NeRF MLP [S2].

1.  **Coarse Sampling:**
    Sample $N_c$ distances $t^c$ uniformly from the near plane $t_n$ to the far plane $t_f$:
    $$
    t^c \sim \mathcal{U}[t_n, t_f], \quad t^c = \text{sort}(\{t^c\})
    $$
    These intervals are converted to conical frustums $(\boldsymbol{\mu}, \boldsymbol{\Sigma})$ [S2].

2.  **Proposal Network Forward Pass:**
    Compute encoded features $\gamma(\boldsymbol{\mu}, \boldsymbol{\Sigma})$ for each coarse interval.
    Query the proposal MLP to get densities $\tau_p$:
    $$
    \tau_p = F_p(\gamma(\boldsymbol{\mu}, \boldsymbol{\Sigma}))
    $$
    Compute coarse alpha compositing weights $w^c$ and rendered color $\mathbf{C}^c$:
    $$
    w_i^c = \left(1 - e^{-\tau_{p,i}(t_{i+1}^c-t_i^c)}\right)e^{-\sum_{j<i} \tau_{p,j}(t_{j+1}^c-t_j^c)}
    $$
    $$
    \mathbf{C}^c = \sum_i w_i^c \mathbf{c}_i \quad (\text{Note: } \mathbf{c}_i \text{ is not predicted by proposal, but weights are used for sampling})
    $$
    *Correction based on [S2]:* The proposal MLP only predicts density. The weights $w^c$ are derived from these densities. The color $\mathbf{C}^c$ is not used for the proposal loss, but the weights $w^c$ are used to guide fine sampling. The NeRF MLP is not queried in the coarse pass for color in the standard description, but the proposal weights are used for the next step [S2].

3.  **Fine Sampling (Online Distillation):**
    Sample $N_f$ distances $t^f$ from the histogram defined by the coarse intervals $t^c$ and coarse weights $w^c$:
    $$
    t^f \sim \text{hist}(\mathbf{t}^c, \mathbf{w}^c), \quad t^f = \text{sort}(\{t^f\})
    $$
    This concentrates samples around high-density regions [S2].

4.  **NeRF Network Forward Pass:**
    Compute encoded features $\gamma(\boldsymbol{\mu}, \boldsymbol{\Sigma})$ for fine intervals.
    Query the NeRF MLP to get densities $\tau_n$ and colors $\mathbf{c}$:
    $$
    (\tau_n, \mathbf{c}) = F_n(\gamma(\boldsymbol{\mu}, \boldsymbol{\Sigma}), \mathbf{d})
    $$
    Compute fine weights $w^f$ and rendered color $\mathbf{C}^f$:
    $$
    w_i^f = \left(1 - e^{-\tau_{n,i}(t_{i+1}^f-t_i^f)}\right)e^{-\sum_{j<i} \tau_{n,j}(t_{j+1}^f-t_j^f)}
    $$
    $$
    \mathbf{C}^f = \sum_i w_i^f \mathbf{c}_i
    $$

5.  **Loss Functions:**
    *   **Reconstruction Loss:** Minimize the mean squared error (MSE) between the rendered fine color and the ground truth $\mathbf{C}^*$, with a weighted contribution from the coarse pass:
        $$
        \mathcal{L}_{\text{recon}} = \sum_{\mathbf{r} \in \mathcal{R}} \frac{1}{10} \|\mathbf{C}^c(\mathbf{r}) - \mathbf{C}^*(\mathbf{r})\|^2 + \|\mathbf{C}^f(\mathbf{r}) - \mathbf{C}^*(\mathbf{r})\|^2
        $$
        *Note: [S2] specifies $\mathcal{L}_{\text{recon}}$ is MSE. The coarse term is weighted by 1/10.*
    *   **Proposal Distillation Loss:** The proposal MLP is supervised by the histogram weights generated by the NeRF MLP, not by image reconstruction. This "online distillation" ensures the proposal network learns the density distribution of the final model [S2]. The specific loss function for this is not explicitly formulaized in the sources, but it involves matching the proposal's density-derived weights to the NeRF's weights [S2].
    *   **Distortion Regularizer:** A novel regularizer is applied to prevent floating artifacts and background collapse. It is designed to work with the continuous weights defined along each mip-NeRF ray [S2]. The exact formula is not provided in the sources, but it penalizes the "center of mass" of the weights being far from the actual geometry [S2].

### Parameters
*   **Number of Frequency Bands ($L$):** Not given in the sources.
*   **Coarse Samples ($N_c$):** Not given in the sources.
*   **Fine Samples ($N_f$):** Not given in the sources.
*   **Near/Far Planes ($t_n, t_f$):** Determined by the camera frustum. For unbounded scenes, $t_f$ is effectively infinite, handled by the contract mapping [S2].

### Edge Cases and Pitfalls
*   **Surface Accuracy:** Mip-NeRF 360 has poor surface accuracy because it uses a density representation without explicit surface geometry [S1].
*   **Unbounded Ambiguity:** Without the contract mapping and distortion regularizer, unbounded scenes suffer from background collapse and floating artifacts [S2].
*   **Aliasing:** Without cone-casting and integrated positional encoding, multi-scale details cause aliasing artifacts [S1].

## Subtopics

### [Zip-NeRF](3030-zip-nerf.md)
Replaces the large MLP with an Instant-NGP grid structure and introduces specific anti-aliasing mechanisms (multisampling, downweighting, and interlevel loss) to handle high-frequency details of grid-based representations.

### [Distortion Loss with Power Transformation](4560-distortion-loss-with-power-transformation.md)
A modification of the mip-NeRF 360 distortion loss that applies a power transformation to metric distance before computing the loss, aggressively penalizing "floaters" near the camera.

## Operating characteristics
*   **Time Complexity:** Training time is approximately 26.8 hours on the Blender dataset, which is $\sim 2\times$ slower than Mip-NeRF and 300% faster than NeRF in Detail's learned proposer approach [S2].
*   **Memory Complexity:** Total model capacity is $\sim 15\times$ larger than Mip-NeRF due to the separate proposal and NeRF MLPs [S2].
*   **Performance:**
    *   NeRF Synthetic (Blender): PSNR = 32.4 +- 0.4 dB [S1].
    *   Mip-NeRF 360 Unbounded Dataset: PSNR = 29.4 +- 0.4 dB [S1].
    *   LLFF Forward-Facing: PSNR = 26.4 +- 0.4 dB [S1].
*   **Strengths:** High quality on unbounded scenes; reduces mean squared error by 57% compared to Mip-NeRF; effective anti-aliasing via cone-casting [S1, S2].
*   **Limitations:** Poor surface accuracy; slow training compared to hash-grid methods like Instant-NGP; high memory usage due to large MLP capacity [S1, S2].
*   **When to Use:** When rendering unbounded scenes with high dynamic range in depth and scale, and surface accuracy is not the primary requirement.
*   **When Not to Use:** When real-time training or fast prototyping is needed (use Instant-NGP); when explicit surface geometry is required (use NeuS or 3DGS).

## Background
Mip-NeRF 360 was introduced in November 2021 as an extension of Mip-NeRF [S2]. It was designed to address the limitations of NeRF and Mip-NeRF in unbounded scenes, where the camera rotates 360 degrees and content exists at any distance [S2]. The method builds on the anti-aliasing techniques of Mip-NeRF and introduces a contract mapping for scene parameterization, online distillation for efficient sampling, and a distortion regularizer for stability [S2]. It is widely used for high-quality novel view synthesis of real-world, unbounded environments [S1].

## Sources
[S1] Neural Rendering Techniques, https://doi.org/10.5281/zenodo.19614729, fetched 2026-10-26
[S2] Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00539, fetched 2026-10-26
[S3] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
