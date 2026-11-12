# GO-Surf


## Overview
GO-Surf is a direct feature grid optimization method for accurate and fast surface reconstruction from RGB-D sequences [S1]. It models the scene with a learned hierarchical feature voxel grid that encapsulates multi-level geometric and appearance local information [S1]. Unlike MLP-based methods, GO-Surf directly optimizes feature vectors in a voxel grid, which are decoded into signed distance field (SDF) and radiance values [S1]. This approach is approximately 60 times faster than MLP-based methods while preserving high-frequency details [S1]. It builds on the hierarchical grid optimization principles of [DirectVoxGO](8134-directvoxgo.md) but specifically targets surface reconstruction via SDF decoding and RGB-D supervision.

## How it works
**Inputs and Outputs:**
- **Inputs:** RGB-D image sequences, camera poses (optimized jointly), and initial scene bounds.
- **Outputs:** Optimized hierarchical feature grids ($V_\theta, W_\beta$), decoder parameters ($\omega, \gamma$), camera poses, and a reconstructed surface (implicit SDF).

**Data Structures:**
1.  **Geometry Feature Grid ($V_\theta$):** A hierarchical structure with $L=4$ levels ($l \in \{0, 1, 2, 3\}$).
    -   Level $l=0$ is the coarsest, $l=3$ is the finest.
    -   Each voxel at any level stores a geometry feature vector of dimension $h_g = 4$.
    -   Total geometry features per voxel across all levels: $4 \times 4 = 16$.
2.  **Color Feature Grid ($W_\beta$):** A single-level feature grid (finest level only).
    -   Each voxel stores a color feature vector of dimension $h_c = 6$.
3.  **Decoders:**
    -   Geometry MLP $f_\omega(\cdot)$: 2 hidden layers, 32 neurons each.
    -   Color MLP $g_\gamma(\cdot)$: 2 hidden layers, 32 neurons each.

**Numbered Steps:**

1.  **Feature Interpolation:** For a 3D point $x \in \mathbb{R}^3$, obtain the geometry feature vector $V_\theta(x)$ by trilinearly interpolating the features from each of the four levels and concatenating them [S1]:
    $$ V_\theta(x) = [V_0(x), V_1(x), V_2(x), V_3(x)] $$
    where $V_l(x)$ is the trilinearly interpolated feature vector at level $l$ [S1].

2.  **SDF Decoding:** Decode the concatenated geometry feature $V_\theta(x)$ into a signed distance value $\phi(x)$ using the geometry MLP $f_\omega$ [S1]:
    $$ \phi(x) = f_\omega(V_\theta(x)) $$

3.  **Color Decoding:** For the same point $x$ and viewing direction $d$, obtain the color feature $W_\beta(x)$ from the finest-level color grid and decode it into color $c$ using the color MLP $g_\gamma$ [S1]:
    $$ c = g_\gamma(W_\beta(x), d) $$

4.  **Ray Sampling:** For a camera ray parameterized by origin $o$ and direction $r$, sample $N$ points $x_i$ along the ray [S1]:
    $$ x_i = o + d_i r, \quad i \in \{1, \dots, N\} $$
    where $d_i$ are the sampled depths [S1].

5.  **Opacity Calculation:** Compute the opacity $\alpha_i$ at each sample point $x_i$ based on the SDF values $\phi(x_i)$ and $\phi(x_{i+1})$ [S1]:
    $$ \alpha_i = \max\left( \frac{\sigma_s(\phi(x_i)) - \sigma_s(\phi(x_{i+1}))}{\sigma_s(\phi(x_i))}, 0 \right) $$
    where $\sigma_s(x) = (1 + e^{-sx})^{-1}$ is a sigmoid function modulated by a learnable parameter $s$ [S1]. The parameter $s$ controls the smoothness of the transition at the surface [S1].

6.  **Volume Rendering:** Compute the accumulated transmittance $T_i$ and weights $w_i$ [S1]:
    $$ T_i = \prod_{j=1}^{i-1} (1 - \alpha_j) $$
    $$ w_i = T_i \alpha_i $$
    Render the expected color $\hat{c}$ and depth $\hat{d}$ [S1]:
    $$ \hat{c} = \sum_{i=1}^{N} w_i c_i $$
    $$ \hat{d} = \sum_{i=1}^{N} w_i d_i $$

**Parameters:**
-   Geometry feature dimension per level: $h_g = 4$ [S1].
-   Color feature dimension: $h_c = 6$ [S1].
-   MLP architecture: 2 hidden layers, 32 neurons each for both decoders [S1].
-   Sigmoid sharpness: Learnable parameter $s$ [S1].
-   Number of samples: $N$ (Not given in the sources).

**Edge Cases and Pitfalls:**
-   **Precision:** Trilinear interpolation requires careful handling of grid boundaries. Points outside the voxel grid bounds must be handled (e.g., clamping or zero-padding) [G].
-   **Degenerate Input:** The opacity formula involves division by $\sigma_s(\phi(x_i))$. If $\sigma_s(\phi(x_i))$ is zero or very close to zero, numerical instability may occur. A small epsilon should be added to the denominator [G].
-   **Surface Smoothness:** The parameter $s$ significantly affects the sharpness of the rendered surface. Too high $s$ may lead to aliasing; too low $s$ may result in blurry surfaces [S1].

## Operating characteristics
-   **Time Complexity:** Significantly faster than MLP-based methods (approx. 60x speedup) due to direct feature grid optimization and shallow decoders [S1].
-   **Memory Complexity:** Depends on the resolution of the 4-level feature grids. Memory usage is higher than single-resolution grids but lower than dense high-resolution grids for equivalent detail.
-   **Strengths:** Fast optimization, high-fidelity surface reconstruction, hole-filling capabilities via multi-level features, direct RGB-D supervision without post-processing [S1].
-   **Limitations:** Requires RGB-D input. Performance depends on the choice of grid resolutions and the learnable parameter $s$ [S1].
-   **When to use:** When fast, high-quality surface reconstruction from RGB-D data is required.
-   **When not to use:** When only RGB input is available (requires depth) or when real-time inference speed is critical and memory is constrained (feature grids can be large).

## Background
GO-Surf was introduced in 2022 by [S1]. It addresses the need for fast and accurate surface reconstruction from RGB-D sequences. Previous methods relied on MLPs, which are slow to optimize. GO-Surf introduces a hybrid scene representation combining multi-level feature grids with shared MLPs for geometry and color prediction. It formulates a smoothness prior on SDF values to improve hole filling and smoothness while preserving details. The method optimizes feature volumes, decoders, and camera poses jointly.

## Sources
[S1] GO-Surf: Neural Feature Grid Optimization for Fast, High-Fidelity RGB-D Surface Reconstruction, https://doi.org/10.1109/3dv57658.2022.00055, fetched 2026-10-29
