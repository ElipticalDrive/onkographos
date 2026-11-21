# Neural Deformable Voxel Grid (NDVG)

## Overview
Neural Deformable Voxel Grid (NDVG) is a fast optimization method for dynamic scene view synthesis that integrates voxel-grid optimization with a deformable radiance field [S1]. It replaces heavy Multi-Layer Perceptron (MLP) computations with fast voxel-grid feature interpolation to enable space-time view synthesis for dynamic scenes captured by a monocular camera [S1]. Unlike static scene applications where occlusion is determined solely by viewing direction, NDVG explicitly models occlusion in dynamic scenes, which is determined by both view direction and motion (time) [S1]. It achieves comparable performance to D-NeRF but trains more than 70× faster [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sequence of images $\{I_t\}_{t=1}^T$ with corresponding camera poses $\{T_t\}_{t=1}^T$ from a monocular camera. A query consists of a 3D point $\mathbf{p}$, viewing direction $\mathbf{d}$, and time instance $t$.
*   **Outputs:** Density $\sigma$ and color $\mathbf{c}$ for the query, which are used to render pixel colors via volume rendering.

**Data Structures:**
The method uses a hybrid explicit-implicit representation consisting of three 3D feature grids and two light-weight MLPs.

1.  **Deformation Feature Grid ($G_d$):**
    *   Layout: Tensor of shape $(N_x, N_y, N_z, N_c)$.
    *   Content: Stores 3D dynamic features for deformation.
    *   Access: Trilinear interpolation.
2.  **Density Grid ($G_\sigma$):**
    *   Layout: Tensor of shape $(N'_x, N'_y, N'_z)$.
    *   Content: Stores density values.
    *   Access: Trilinear interpolation followed by softplus activation.
3.  **Color Feature Grid ($G_c$):**
    *   Layout: Tensor of shape $(H', W', D', N'_c)$.
    *   Content: Stores view-dependent color features.
    *   Access: Trilinear interpolation.
4.  **Deformation MLP ($F_{\theta^d_1}$):**
    *   Architecture: 4 layers, width 64.
    *   Input: Concatenation of positional encoding of $\mathbf{p}$, positional encoding of $t$, and interpolated feature $\hat{\mathbf{f}}^d$.
    *   Output: Deformation offset $\Delta \mathbf{p}$ and occlusion weight $\hat{w}^{occ}$.
5.  **Canonical MLP ($F_{\theta^c_2}$):**
    *   Architecture: 3 layers, width 128.
    *   Input: Concatenation of positional encoding of canonical position $\mathbf{p}'$, viewing direction $\mathbf{d}$, and interpolated feature $\hat{\mathbf{f}}^c$.
    *   Output: Color $\mathbf{c}$.

**Parameters:**
*   **Positional Encoding:** Frequency 5 for position and time, 4 for direction [S1].
*   **Canonical Time ($t_{can}$):** The reference time for the canonical space [S1].
*   **Grid Resolutions:**
    *   Coarse Stage: Deformation module $1,664k$ voxels; Canonical module $1,024k$ voxels. Color grid dimension is 3 (RGB) [S1].
    *   Fine Stage: Deformation module $190^3$ voxels; Canonical module $160^3$ voxels [S1].
*   **Training Iterations:**
    *   Full Resolution: 10k iterations for coarse stage, 20k for fine stage [S1].
    *   Half Resolution: 5k iterations for coarse stage, 10k for fine stage [S1].
*   **Optimization:** Adam optimizer, 8,192 rays sampled per iteration [S1].

**Algorithm Steps:**

1.  **Deformation Query:** For a query point $\mathbf{p}$ at time $t$:
    *   Perform trilinear interpolation on $G_d$ to get feature $\hat{\mathbf{f}}^d = \text{interp}(\mathbf{p}, G_d)$ [S1].
    *   If $t = t_{can}$, set $\Delta \mathbf{p} = \mathbf{0}$ and $\hat{w}^{occ} = 1$ [S1].
    *   Otherwise, input $\mathbf{p}$, $t$, and $\hat{\mathbf{f}}^d$ into $F_{\theta^d_1}$ to regress $\Delta \mathbf{p}$ and $\hat{w}^{occ}$ [S1].
    *   Note: $\mathbf{p}$ and $t$ are passed through positional encoding before entering the MLP [S1].

2.  **Canonical Position:** Compute the canonical position $\mathbf{p}' = \mathbf{p} + \Delta \mathbf{p}$ [S1].

3.  **Density Query:**
    *   Perform trilinear interpolation on $G_\sigma$ to get density value $v_\sigma = \text{interp}(\mathbf{p}', G_\sigma)$ [S1].
    *   Apply softplus activation: $\sigma = \text{softplus}(v_\sigma)$ [S1].

4.  **Color Query:**
    *   Perform trilinear interpolation on $G_c$ to get feature $\hat{\mathbf{f}}^c = \text{interp}(\mathbf{p}', G_c)$ [S1].
    *   Input $\mathbf{p}'$, $\mathbf{d}$, and $\hat{\mathbf{f}}^c$ into $F_{\theta^c_2}$ to get color $\mathbf{c}$ [S1].
    *   Note: $\mathbf{p}'$ and $\mathbf{d}$ are passed through positional encoding before entering the MLP [S1].
    *   In the coarse stage, $G_c$ has dimension 3, and the MLP is bypassed or simplified to directly output RGB [S1].

5.  **Occlusion Filtering:**
    *   Filter density and color using the occlusion mask: $\sigma' = \sigma \times \hat{w}^{occ}$ and $\mathbf{c}' = \mathbf{c} \times \hat{w}^{occ}$ [S1].
    *   The occlusion mask $\hat{w}^{occ}$ is 0 if a point is empty in observation space but occupied in canonical space, preventing artifacts where static empty points map to occupied locations [S1].

6.  **Volume Rendering:**
    *   Render ray color $\hat{C}(\mathbf{r})$ using standard volume rendering equation with filtered density $\sigma'$ and color $\mathbf{c}'$ [S1].
    *   For a ray $\mathbf{r}$ with $K$ sampled points:
        $$ \hat{C}(\mathbf{r}) = \sum_{k=1}^{K} T_k \alpha_k \mathbf{c}'_k $$
        $$ T_k = \exp \left( -\sum_{j=1}^{k-1} \sigma'_j \delta_j \right), \quad \alpha_k = 1 - \exp(-\sigma'_k \delta_k) $$
        where $\delta_k$ is the distance between adjacent samples [S1].

7.  **Loss Calculation:**
    *   Compute total loss $L$ as a weighted sum of components [S1]:
        $$ L = L_{photo} + w_{ptc} L_{ptc} + w_{bg} L_{bg} + w_{d\_norm} L_{d\_norm} + w_{d\_tv} L_{d\_tv} $$
    *   **Photometric Loss:**
        $$ L_{photo} = \frac{1}{|\mathcal{R}|} \sum_{\mathbf{r} \in \mathcal{R}} \| \hat{C}(\mathbf{r}) - C(\mathbf{r}) \|_2^2 $$
        where $\mathcal{R}$ is the set of rays in a batch [S1].
    *   **Sampled Point Color Supervision ($L_{ptc}$):** Optimizes color of sampled points with top $N$ weights contributing most to rendered color [S1].
    *   **Background Entropy Loss ($L_{bg}$):** Encourages points to concentrate on foreground or background [S1].
    *   **Deformation L1 Regularization:**
        $$ L_{d\_norm} = \frac{1}{|\mathcal{R}|} \sum_{\mathbf{r} \in \mathcal{R}} \sum_{i=1}^{K} \| \Delta \mathbf{p}_i \|_1 $$
        Regularizes deformation based on the prior that most points are static [S1].
    *   **Total Variation Loss on Deformation Grid:**
        $$ L_{d\_tv} = \frac{1}{|\mathcal{V}|} \sum_{\mathbf{v} \in \mathcal{V}} \sum_{c \in [N_c]} \sqrt{\Delta_x^2(\mathbf{v},c) + \Delta_y^2(\mathbf{v},c) + \Delta_z^2(\mathbf{v},c)} $$
        where $\Delta_x^2(\mathbf{v},c)$ is the squared difference between the $c$-th element in voxel $\mathbf{v}(i,j,k)$ and voxel $\mathbf{v}(i+1,j,k)$ [S1].

8.  **Training Strategy:**
    *   **Progressive Training:** Start training with images close to canonical time, then progressively add images with further time steps to reduce learning difficulty as deformation magnitude increases [S1].
    *   **Coarse-to-Fine Optimization:**
        1.  Optimize a coarse model (no view-dependence, color grid dim 3) to roughly recover deformation and geometry [S1].
        2.  Use coarse model to locate object region and filter empty space (bounding box and empty point filtering) [S1].
        3.  Optimize fine model with higher resolution, initialized with coarse weights [S1].

**Pitfalls and Edge Cases:**
*   **Occlusion Artifacts:** If an empty point in observation space is static ($\Delta \mathbf{p}=0$) but maps to an occupied location in canonical space, it retrieves non-zero density/color, causing artifacts. This is solved by the explicit occlusion mask $\hat{w}^{occ}$ [S1].
*   **Inefficient 4D Grid:** Directly storing offsets in a 4D feature grid (3 for $\mathbf{p}$, 1 for $t$) is inefficient. NDVG uses a 3D feature grid and a light-weight MLP to decode the feature instead [S1].
*   **Resolution Sensitivity:** Deformation is more sensitive to resolution than radiance (density and color), so the deformation module grid resolution is set higher than the canonical module [S1].

## Operating characteristics
*   **Speed:** NDVG achieves comparable performance to D-NeRF using only 20 minutes for training, which is more than 70× faster than D-NeRF [S1].
*   **Memory:** Uses voxel grids which are generally more memory-efficient than large MLPs for high-resolution scenes, though grid resolution directly impacts memory usage [S1].
*   **Strengths:** Fast optimization, efficient space-time view synthesis for dynamic scenes, explicit occlusion handling for monocular input [S1].
*   **Limitations:** Performance depends on the quality of the deformation grid and MLP; may struggle with highly complex deformations if grid resolution is insufficient [S1].
*   **When to use:** When fast training of dynamic scenes from monocular video is required [S1].
*   **When not to use:** When multi-view data is available and DeVRF is preferred, or when image quality refinement via look-up tables (V4D) is prioritized over training speed [S1].

## Background
NDVG was proposed by X. Guo et al. in the paper "Neural Deformable Voxel Grid for Fast Optimization of Dynamic View Synthesis" [S1]. It is based on the canonical scene framework where a deformation module maps observation space to canonical space, and volume rendering is performed in the canonical space [S1]. It builds on voxel-grid optimization techniques used in static radiance fields [S1].

## Sources
[S1] Neural Deformable Voxel Grid for Fast Optimization of Dynamic View Synthesis, https://doi.org/10.1007/978-3-031-26319-4_27, fetched 2026-10-26
