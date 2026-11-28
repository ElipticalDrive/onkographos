# V4D (Voxel for 4D Novel View Synthesis)
## Overview
V4D is a voxel-based architecture for 4D novel view synthesis in dynamic scenes that models the 4D neural radiance field directly without requiring a canonical space [S1]. It addresses the limited representation capacity of MLPs under single-view constraints by using separate 3D voxel grids for density and texture, combined with a conditional positional encoding module and a look-up table (LUT) refinement module to preserve high-frequency details and prevent over-smoothing [S1]. Unlike continuous neural radiance fields that require per-pixel optimization, V4D pre-computes a shared volumetric representation, offering competitive reconstruction quality with shorter inference time than TiNeuVox [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A single-view video sequence (providing spatial coordinates $\mathbf{x}=(x,y,z)$, time index $t$, and view direction $(\theta, \phi)$) and ground-truth RGB images for supervision [S1].
*   **Output:** Rendered RGB images $\hat{C}(\mathbf{r})$ for novel viewpoints and time steps [S1].

**Data Structures:**
1.  **Density Volume:** A 3D voxel grid initialized with dimensions $160 \times 160 \times 160 \times 12$ storing features $F_{den}$ [S1].
2.  **Texture Volume:** A 3D voxel grid initialized with dimensions $160 \times 160 \times 160 \times 12$ storing features $F_{tex}$ [S1].
3.  **Basic LUTs:** $M=5$ learnable 3D look-up tables, each with resolution $33 \times 33 \times 33$ [S1].
4.  **MLPs:** Density MLP (5 layers), Texture MLP (5 layers), and Weight Prediction MLP (10 layers) [S1].

**Numbered Steps:**

1.  **Voxel Sampling:** For a ray $\mathbf{r}$ sampled at $N$ points along its path, retrieve the feature vectors $F_{den}$ and $F_{tex}$ from the density and texture volumes at the 3D spatial location $(x, y, z)$ using trilinear interpolation [S1].
2.  **Conditional Positional Encoding (CPE):** Apply CPE to the sampled texture features $F_{tex}$ to recover high-frequency details lost by regularization [S1]. The encoding is defined as:
    $$
    \gamma(p_v) = \left( \sin\left(2^\pi p_v + \frac{2\pi}{L-1}t\right), \cos\left(2^\pi p_v + \frac{2\pi}{L-1}t\right) \right)
    $$
    where $p_v$ is the sampled feature vector, $L=5$ is the frequency order, and $t$ is the time index [S1].
    *   **Pitfall:** CPE is **not** applied to the density features $F_{den}$. The density volume must be initialized to zero for correct initial volume rendering; applying CPE to density features breaks this initialization and causes learning collapse [S1].
    *   Let the encoded texture feature be $\hat{F}_{tex}$ [G]. The density feature remains $F_{den}$.
3.  **Feature Concatenation:** Concatenate the time index $t$, the 3D location $(x, y, z)$, the view direction $(\theta, \phi)$, and the features ($F_{den}$ for density, $\hat{F}_{tex}$ for texture) into a single vector [S1].
4.  **Field Estimation:**
    *   Feed the concatenated vector into the 5-layer Density MLP to predict volume density $\sigma$ [S1].
    *   Feed the concatenated vector into the 5-layer Texture MLP to predict RGB color $\mathbf{c}$ [S1].
5.  **Volume Rendering:** Compute the coarse RGB color $\hat{C}(\mathbf{r})$ for the ray using the standard emission-absorption model:
    $$
    \hat{C}(\mathbf{r}) = \sum_{i=1}^{N} T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
    $$
    where $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the transmittance, $\delta_i = t_{i+1} - t_i$ is the step size, and $N$ is the number of sampled points [S1].
6.  **Pseudo-Surface Depth Estimation:** Compute the expected depth $\hat{d}$ for the ray to guide the refinement module:
    $$
    \hat{d} = \sum_{i=1}^{N} T_i (1 - \exp(-\sigma_i \delta_i)) t_i
    $$
    where $t_i$ represents the depth coordinate of the $i$-th sample point [S1].
7.  **LUT Refinement (Pixel-Level):**
    *   Treat the coarse RGB value $\hat{C}(\mathbf{r})$ as a spatial coordinate in the RGB space [S1].
    *   Perform trilinear interpolation on each of the $M=5$ basic LUTs using this coordinate to obtain interpolated RGB values $\mathbf{r}_m$ for $m=1 \dots M$ [S1].
    *   Use the pseudo-surface depth $\hat{d}$ (and potentially spatial coordinates) as input to the 10-layer Weight Prediction MLP to predict spatial-aware weights $w_m$ for each basic LUT [S1].
    *   Combine the interpolated values using the predicted weights to produce the refined RGB color [S1]. This process is repeated for $Z=3$ recurrent iterations [S1].

**Loss Function:**
The total loss is a weighted sum of four components:
$$
Loss = w_1 L_{rgb} + w_2 L_{pt\_rgb} + w_3 L_{bg} + w_4 L_{TV}
$$
*   **RGB Loss ($L_{rgb}$):** Mean squared error between rendered and ground truth colors [S1].
    $$
    L_{rgb} = \frac{1}{R} \sum_{r \in R} |\hat{C}(r) - C(r)|^2
    $$
    where $R$ is the set of training rays [S1].
*   **Point-wise RGB Loss ($L_{pt\_rgb}$):** Encourages consistency at sampled points [S1].
    $$
    L_{pt\_rgb} = \frac{1}{R} \sum_{r \in R} \sum_{i=1}^{K} (T_i a_i \hat{C}(r) - C(r))^2
    $$
    where $K$ is the number of queried points in the texture volume, $T_i$ is accumulated transmittance, and $a_i$ is the probability of termination at point $i$ [S1].
*   **Background Loss ($L_{bg}$):** Encourages foreground/background separation by penalizing intermediate transmittance values [S1].
    $$
    L_{bg} = -T_{K+1} \log(T_{K+1}) - (1 - T_{K+1}) \log(1 - T_{K+1})
    $$
    where $T_{K+1}$ is the transmittance after the last sample [S1].
*   **Total Variation Loss ($L_{TV}$):** Applied directly to the voxel grids of the density and texture volumes to prevent over-fitting and maintain geometry [S1]. It is **not** applied to the LUTs [S1].

**Parameters:**
*   **Voxel Resolution:** $160 \times 160 \times 160$ [S1].
*   **Feature Dimension:** 12 [S1].
*   **MLP Layers:** 5 for density/texture fields, 10 for weight prediction [S1].
*   **CPE Frequency Order ($L$):** 5 [S1].
*   **Basic LUTs ($M$):** 5 [S1].
*   **LUT Resolution:** $33 \times 33 \times 33$ [S1].
*   **Recurrent Iterations ($Z$):** 3 [S1].
*   **Learning Rates:**
    *   Voxel grids (density/texture): 0.1 [S1].
    *   Density/Texture MLPs: $1 \times 10^{-3}$ [S1].
    *   LUTs refinement MLPs and voxels: $1 \times 10^{-4}$ [S1].
*   **Optimizer:** Adam [S1].
*   **Batch Size:** 8,196 rays (first stage), 4,196 rays (second stage) [S1].
*   **Loss Weights:** $w_1=1.0, w_2=0.01, w_3=0.001, w_4=0.1$ [S1].
*   **TV Loss Decay:** Exponential decay factor of 0.005 applied to $w_4$ during training to alleviate over-smoothing [S1].
*   **Training Iterations:** 250k for datasets [23] and 100k for datasets [21] [S1].

**Pitfalls and Edge Cases:**
*   **Over-smoothing:** Total variation loss causes blurring and loss of high-frequency details [S1]. This is mitigated by the CPE module and the LUT refinement module [S1].
*   **Initialization Collapse:** Applying CPE to density features breaks the zero-initialization rule required for stable volume rendering at the start of training [S1]. CPE must only be applied to texture features [S1].
*   **Spatial Awareness:** Using low-resolution images to predict LUT weights loses 3D spatial awareness [S1]. Using pseudo-surface depth via an MLP preserves this awareness [S1].

## Subtopics

### [MAV3D](11997-mav3d.md)
MAV3D synthesizes time-varying 3D scenes directly from text prompts by optimizing a 4D neural radiance field (4D-NeRF) under the guidance of a pretrained text-to-video diffusion model. It is distinguished from its parent [V4D](8762-v4d-voxel-for-4d-novel-view-synthesis.md) by its data-free supervision strategy, which relies solely on text and video priors rather than multi-view video inputs, and from its sibling [Explicit Neural View Synthesis (Voxel-based Neural Rendering)](14921-explicit-neural-view-synthesis-voxel-based-neural-rendering.md) by its use of a hexplane-based hybrid representation for efficient spacetime encoding.

### [Explicit Neural View Synthesis (Voxel-based Neural Rendering)](14921-explicit-neural-view-synthesis-voxel-based-neural-rendering.md)
Explicit Neural View Synthesis (ENR) is a novel view synthesis method that explicitly encodes sparse source observations into a volumetric representation (RGB$\alpha$ volume) to enable amortized rendering, avoiding per-pixel ray sampling and scene-specific optimization. Unlike continuous neural radiance fields such as NeRF [26], which require independent ray integrals for each pixel and prevent amortization across views, ENR pre-computes a shared RGB$\alpha$ volume that can be rendered for any target view via geometric transformations. This approach offers rendering speeds over 400x faster than continuous baselines while maintaining comparable or superior reconstruction quality.

## Operating characteristics
*   **Time and Memory Complexity:** Training requires 250k iterations for datasets [23] and 100k for datasets [21] [S1]. Memory usage consists of two $160^3 \times 12$ voxel grids and five $33^3$ LUTs [S1].
*   **Measured Performance:** V4D achieves state-of-the-art performance on synthetic dynamic scenes (e.g., PSNR 33.72, SSIM 0.98 on datasets [23]) and competitive performance on real dynamic scenes [S1]. It outperforms D-NeRF, NDVG, and TiNeuVox in PSNR/SSIM [S1].
*   **Scaling:** The method scales to 4K resolution in related real-time variants but standard V4D focuses on single-view dynamic scene synthesis [S1].
*   **Strengths and Limitations:** Strengths include no canonical space requirement, plug-and-play LUT refinement, and high-quality dynamic novel view synthesis from single-view video [S1]. Limitations include longer training time compared to canonical-space methods and potential for over-smoothing if TV loss is not balanced with CPE/LUT refinement [S1].
*   **When to Use:** Use V4D when synthesizing novel views of dynamic scenes from single-view video without requiring a canonical space or multi-view constraints [S1]. Avoid it when faster training convergence is prioritized over inference speed or when multi-view inputs are available [S1].

## Background
V4D was proposed to address the limited representation capacity of MLPs under single-view constraints, specifically the lack of multi-view constraints [S1]. It builds on the principles of differentiable volume rendering [G] and extends 3D voxel-based representations to 4D by modeling density and texture fields separately [S1]. The method was evaluated on 8 synthetic dynamic scene datasets and 4 real scene datasets, demonstrating superiority over existing single-view dynamic novel view synthesis methods [S1].

## Sources
[S1] V4D: Voxel for 4D Novel View Synthesis, https://doi.org/10.1109/tvcg.2023.3312127, fetched 2026-10-26
