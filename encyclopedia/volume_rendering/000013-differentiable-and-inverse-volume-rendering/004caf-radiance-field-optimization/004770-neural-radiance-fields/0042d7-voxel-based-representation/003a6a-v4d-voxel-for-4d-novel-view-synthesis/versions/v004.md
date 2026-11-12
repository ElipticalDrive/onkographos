# V4D (Voxel for 4D Novel View Synthesis)

## Overview
V4D is a voxel-based architecture for 4D novel view synthesis in dynamic scenes that models the 4D neural radiance field directly from single-view video, eliminating the need for a canonical space or explicit deformation fields [S1]. It distinguishes itself from sibling methods like [FastNeRF](7183-fastnerf.md) and [VoxGRAF](12633-voxgraf.md) by using a dual-voxel structure (separate density and texture volumes) combined with a plug-and-play Look-Up Table (LUT) refinement module to recover high-frequency details lost by total variation regularization [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of single-view images (video) representing a dynamic scene.
*   **Output:** Rendered RGB images for novel viewpoints and time steps.

**Data Structures:**
1.  **Density Volume:** A 3D voxel grid of size $160 \times 160 \times 160$ with 12 feature channels. Initialized to zero [S1].
2.  **Texture Volume:** A 3D voxel grid of size $160 \times 160 \times 160$ with 12 feature channels. Initialized to zero [S1].
3.  **Basic LUTs:** $M=5$ learnable 3D lookup tables, each of resolution $33 \times 33 \times 33$, mapping coarse RGB values to refined RGB values. One LUT is initialized as the identity matrix; the remaining $M-1$ are initialized to zero [S1].

**Numbered Steps:**

1.  **Ray Sampling:** For a camera ray $\mathbf{r}$ at time $t$, sample $N$ points $\mathbf{x}_i = (x_i, y_i, z_i)$ along the ray. Compute distances $\delta_i = t_{i+1} - t_i$ [S1].
2.  **Feature Sampling:** Trilinearly interpolate features from the Density Volume ($F_{den}$) and Texture Volume ($F_{tex}$) at each $\mathbf{x}_i$ [S1].
3.  **Conditional Positional Encoding (CPE):** Apply CPE to the texture features $F_{tex}$ to recover high-frequency details. Do **not** apply CPE to density features $F_{den}$ to preserve zero-initialization [S1].
    $$
    \gamma(p_v) = \left( \sin\left(2^\pi p_v + \frac{2\pi}{L-1}t\right), \cos\left(2^\pi p_v + \frac{2\pi}{L-1}t\right) \right)
    $$
    where $p_v$ is the sampled feature vector, $L=5$ is the frequency order, and $t$ is the time index [S1].
4.  **Concatenation:** Concatenate the time index $t$, 3D location $(x, y, z)$, view direction $(\theta, \phi)$, and the processed features ($F_{den}$ and $\hat{F}_{tex}$) [S1].
5.  **MLP Decoding:**
    *   Feed the concatenated vector into a 5-layer MLP to predict volume density $\sigma_i$ [S1].
    *   Feed the concatenated vector into a separate 5-layer MLP to predict RGB color $\mathbf{c}_i$ [S1].
6.  **Volume Rendering:** Compute coarse RGB $\hat{C}(\mathbf{r})$ using standard volume rendering [S1]:
    $$
    \hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
    $$
    where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ [S1].
7.  **Pseudo-Surface Depth Estimation:** Compute depth $\hat{d}$ for spatial awareness in the refinement module [S1]:
    $$
    \hat{d} = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) t_i
    $$
8.  **LUT Refinement (Plug-and-Play):**
    *   Use a 10-layer MLP to predict spatial-aware weights $\mathbf{w}$ based on the pseudo-surface depth $\hat{d}$ [S1].
    *   Perform trilinear interpolation on the $M$ basic LUTs using the coarse RGB $\hat{C}(\mathbf{r})$ as coordinates to get interpolated colors [S1].
    *   Combine interpolated colors using weights $\mathbf{w}$ to produce refined RGB [S1].
    *   Repeat this refinement process for $Z=3$ iterations [S1].

**Parameters:**
*   Voxel Grid Resolution: $160 \times 160 \times 160$ [S1].
*   Voxel Channels: 12 [S1].
*   MLP Depth (Density/Texture): 5 layers [S1].
*   MLP Depth (Weight Prediction): 10 layers [S1].
*   CPE Frequency Order ($L$): 5 [S1].
*   Number of Basic LUTs ($M$): 5 [S1].
*   LUT Resolution: $33 \times 33 \times 33$ [S1].
*   Refinement Iterations ($Z$): 3 [S1].
*   Training Iterations: 250k (synthetic datasets), 100k (real datasets) [S1].
*   Batch Size: 8,196 rays (Stage 1), 4,196 rays (Stage 2) [S1].
*   Optimizer: Adam [S1].
*   Learning Rates: Voxel (0.1), Density/Texture MLPs (1e-3), LUT MLPs/Voxel (1e-4) [S1].
*   Loss Weights: $w_1=1.0$ ($L_{rgb}$), $w_2=0.01$ ($L_{pt\_rgb}$), $w_3=0.001$ ($L_{bg}$), $w_4=0.1$ ($L_{TV}$) [S1].
*   TV Loss Decay: Exponential decay factor 0.005 [S1].

**Edge Cases and Pitfalls:**
*   **Learning Collapse:** Applying CPE to density features breaks the zero-initialization rule required for stable volume rendering at the start of training. CPE must only be applied to texture features [S1].
*   **Over-smoothing:** Total Variation (TV) loss prevents overfitting but causes blurring. This is mitigated by CPE and LUT refinement [S1].
*   **Spatial Awareness:** Using low-resolution images to predict LUT weights loses 3D spatial context. Using pseudo-surface depth via an MLP preserves this awareness [S1].

## Subtopics

### MAV3D
Synthesizes time-varying 3D scenes directly from text by optimizing a 4D neural radiance field under guidance from a pretrained text-to-video diffusion model [S2]. It differs from V4D by using a hexplane-based 4D representation for efficient spacetime encoding rather than 3D voxel grids, and employs a multi-stage static-to-dynamic training scheme with temporal regularization [S2]. It utilizes a data-free supervision strategy using only text and video priors, requiring no 4D datasets [S2].

## Operating characteristics
*   **Time Complexity:** Training is computationally intensive due to the lack of a canonical space adaptation, requiring more iterations (250k) compared to canonical-space methods like D-NeRF [S1]. Inference is faster than TiNeuVox because it avoids MLP-based sub-networks for predicting position offsets [S1].
*   **Memory Complexity:** Uses two $160^3$ voxel grids (density and texture) plus LUTs. This is more memory-intensive than implicit MLP-only methods but less than full 4D voxel grids [S1].
*   **Performance:** Achieves state-of-the-art PSNR/SSIM on synthetic dynamic scenes (outperforming D-NeRF, NDVG, TiNeuVox) [S1]. Competitive on real dynamic scenes [S1].
*   **Strengths:** No canonical space required; plug-and-play LUT refinement; effective high-frequency detail recovery [S1].
*   **Limitations:** Longer training time; relies on TV loss which requires careful tuning to avoid over-smoothing [S1].

## Background
V4D was proposed to address the limitations of MLP-only NeRF representations in dynamic scenes, where global optimization is slow and capacity is limited without multi-view constraints [S1]. It extends 3D voxel-based architectures to 4D by directly mapping 6D attributes (space, time, view) to density and color [S1]. The method was evaluated on 8 synthetic and 4 real dynamic scene datasets [S1].

## Sources
[S1] V4D: Voxel for 4D Novel View Synthesis, https://doi.org/10.1109/tvcg.2023.3312127, fetched 2026-10-26
[S2] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
