# Neural Deformable Voxel Grid (NDVG)

## Overview
Neural Deformable Voxel Grid (NDVG) is a fast deformable radiance field method for dynamic scene view synthesis that integrates voxel-grid optimization with a deformable radiance field [S1]. It replaces heavy Multi-Layer Perceptron (MLP) computations with fast voxel-grid feature interpolation to enable space-time view synthesis for dynamic scenes captured by a monocular camera [S1]. Unlike static scene applications where occlusion is determined solely by viewing direction, NDVG explicitly models occlusion in dynamic scenes, which is determined by both view direction and motion (time) [S1]. It achieves comparable performance to D-NeRF but trains more than 70× faster [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** An image sequence $\{I_t\}_{t=1}^T$ with corresponding camera poses $\{T_t\}_{t=1}^T$ of a dynamic scene captured by a monocular camera. A query 3D point $p$, view direction $d$, and time instance $t$.
*   **Outputs:** Estimated density $\sigma$ and color $c$ for the query point, which are used to render the pixel color $C(r)$ via volume rendering [S1].

**Data Structures:**
The method uses a hybrid explicit-implicit representation consisting of three 3D feature grids and two light-weight MLPs [S1].

1.  **Deformation Feature Grid ($G_d$):**
    *   **Layout:** Tensor of shape $(N_x, N_y, N_z, N_c)$.
    *   **Content:** Stores 3D dynamic features. Each voxel contains a feature vector of dimension $N_c$ [S1].
    *   **Access:** Trilinear interpolation $\text{interp}(p, G_d)$ returns a vector in $\mathbb{R}^{N_c}$ [S1].

2.  **Density Grid ($G_\sigma$):**
    *   **Layout:** Tensor of shape $(N'_x, N'_y, N'_z)$.
    *   **Content:** Stores density values for the canonical scene geometry [S1].
    *   **Access:** Trilinear interpolation $\text{interp}(p', G_\sigma)$ returns a scalar [S1].

3.  **Color Feature Grid ($G_c$):**
    *   **Layout:** Tensor of shape $(H', W', D', N'_c)$.
    *   **Content:** Stores view-dependent color features. Each voxel contains a feature vector of dimension $N'_c$ [S1].
    *   **Access:** Trilinear interpolation $\text{interp}(p', G_c)$ returns a vector in $\mathbb{R}^{N'_c}$ [S1].

4.  **Deformation MLP ($F_{\theta^d_1}$):**
    *   **Architecture:** 4 layers, width 64 [S1].
    *   **Input:** Concatenation of positional encoding of $p$, positional encoding of $t$, and deformation feature $f^d$.
    *   **Output:** Deformation offset $\Delta p$ and occlusion weight $w^{occ}$ [S1].

5.  **Canonical MLP ($F_{\theta^c_2}$):**
    *   **Architecture:** 3 layers, width 128 [S1].
    *   **Input:** Concatenation of positional encoding of $p'$, positional encoding of $d$, and color feature $f^c$.
    *   **Output:** Color $c$ [S1].

**Parameters:**
*   **Positional Encoding:** Frequency 5 for position and time, 4 for direction [S1].
*   **Coarse Stage Grid Voxel Count:** Deformation module 1,664k voxels; Canonical module 1,024k voxels [S1].
*   **Fine Stage Grid Voxel Count:** Deformation module $190^3$ voxels; Canonical module $160^3$ voxels [S1].
*   **Coarse Stage Color Grid Dimension:** 3 (RGB) [S1].
*   **Training Iterations:** 10k coarse / 20k fine (full res); 5k coarse / 10k fine (half res) [S1].
*   **Optimizer:** Adam [S1].
*   **Rays per Iteration:** 8,192 [S1].

**Algorithm Steps:**

1.  **Deformation Query:**
    For a query point $p$ at time $t$:
    a. Perform trilinear interpolation on $G_d$ to get feature $f^d = \text{interp}(p, G_d)$ [S1].
    b. If $t = t_{can}$ (canonical time), set $\Delta p = 0$ [S1].
    c. If $t \neq t_{can}$, input $p$, $t$, and $f^d$ into $F_{\theta^d_1}$ to regress $\Delta p$ and $w^{occ}$ [S1].
    $$
    \begin{cases}
    \Delta p = F_{\theta^d_1}(p, t, f^d) \\
    w^{occ} = \text{output}_2(F_{\theta^d_1}(p, t, f^d))
    \end{cases}
    $$
    [S1]

2.  **Canonical Position:**
    Compute the canonical position $p'$:
    $$ p' = p + \Delta p $$
    [S1]

3.  **Density Query:**
    For canonical point $p'$:
    a. Perform trilinear interpolation on $G_\sigma$: $v_\sigma = \text{interp}(p', G_\sigma)$ [S1].
    b. Apply softplus activation:
    $$ \sigma = \text{softplus}(v_\sigma) $$
    [S1]

4.  **Color Query:**
    For canonical point $p'$ and view direction $d$:
    a. Perform trilinear interpolation on $G_c$: $f^c = \text{interp}(p', G_c)$ [S1].
    b. Input $p'$, $d$, and $f^c$ into $F_{\theta^c_2}$ to get color $c$:
    $$ c = F_{\theta^c_2}(p', d, f^c) $$
    [S1]

5.  **Occlusion Filtering:**
    Filter density and color using the occlusion mask $w^{occ}$:
    $$ \sigma' = \sigma \times w^{occ} $$
    $$ c' = c \times w^{occ} $$
    [S1]
    *Note:* If a point is empty in observation space but occupied in canonical space, $w^{occ}$ should be 0 to prevent artifacts [S1].

6.  **Volume Rendering:**
    Render ray color $C(r)$ using the standard volume rendering equation with filtered density $\sigma'$ and color $c'$ [S1]:
    $$ C(r) = \sum_{k=1}^{K} T(w_k) \alpha(w_k) c'(w_k) $$
    $$ T(w_k) = \exp \left( -\sum_{j=1}^{k-1} \sigma'(t_j) \delta_j \right) $$
    $$ \alpha(w_k) = 1 - \exp(-\sigma'(w_k) \delta_k) $$
    where $K$ is the number of sampled points, $\delta_k$ is the distance between adjacent samples, and $t_j$ are the sample points [S1].

**Loss Functions:**
The total loss $L$ is a weighted sum of the following components [S1]:

1.  **Photometric Loss ($L_{photo}$):** Mean squared error between estimated ray color $\hat{C}(r)$ and ground truth $C(r)$:
    $$ L_{photo} = \frac{1}{|\mathcal{R}|} \sum_{r \in \mathcal{R}} \| \hat{C}(r) - C(r) \|_2^2 $$
    where $\mathcal{R}$ is the set of rays sampled in one batch [S1].

2.  **Deformation L1 Regularization ($L_{d\_norm}$):** Encourages most points to be static:
    $$ L_{d\_norm} = \frac{1}{|\mathcal{R}|} \sum_{r \in \mathcal{R}} \sum_{i=1}^{K} \| \Delta p_i \|_1 $$
    where $i$ indexes sampled points $K$ on ray $r$ [S1].

3.  **Total Variation Loss on Deformation Grid ($L_{d\_tv}$):** Smooths voxel features:
    $$ L_{d\_tv} = \frac{1}{|\mathcal{V}|} \sum_{v \in \mathcal{V}} \sum_{c \in [N_c]} \sqrt{\Delta_x^2(v,c) + \Delta_y^2(v,c) + \Delta_z^2(v,c)} $$
    where $\Delta_x^2(v,c)$ is the squared difference between the $c$-th element in voxel $v(i,j,k)$ and $v(i+1,j,k)$ [S1].

4.  **Point Color Supervision ($L_{ptc}$) and Background Entropy Loss ($L_{bg}$):** Used as in [48] (not detailed in source text, but included in total loss) [S1].

**Total Loss:**
$$ L = L_{photo} + w_{ptc} L_{ptc} + w_{bg} L_{bg} + w_{d\_norm} L_{d\_norm} + w_{d\_tv} L_{d\_tv} $$
where $w$ terms are balancing weights [S1].

**Training Procedure:**
1.  **Progressive Training:** Start training with images close to the canonical time $t_{can}$, then progressively add images with further time steps to reduce learning difficulty as deformation magnitude increases [S1].
2.  **Coarse-to-Fine Optimization:**
    a. **Coarse Stage:** Optimize a coarse model without view-dependence (color grid dimension 3) to roughly recover deformation and geometry. This locates object regions and filters empty space [S1].
    b. **Fine Stage:** Optimize a fine model with higher resolution, initialized with coarse weights. Apply empty space filtering strategies (bounding box, empty point filtering) to speed up training [S1].

**Pitfalls and Edge Cases:**
*   **Occlusion Artifacts:** If an empty point in observation space is static ($\Delta p=0$) but maps to an occupied location in canonical space, it retrieves non-zero density/color, causing artifacts. This is solved by the occlusion mask $w^{occ}$ [S1].
*   **Inefficient Storage:** Directly storing offsets in a 4D feature grid (3 for $p$, 1 for $t$) is inefficient. NDVG uses a 3D feature grid and a light-weight MLP instead [S1].
*   **Resolution Sensitivity:** Deformation is more sensitive to resolution than radiance; thus, the deformation module grid resolution is set higher than the canonical module [S1].

## Operating characteristics
*   **Speed:** NDVG achieves comparable performance to D-NeRF using only 20 minutes for training, which is more than 70× faster than D-NeRF [S1].
*   **Memory/Complexity:** Uses voxel-grid representation which is generally more memory efficient and faster to query than dense MLPs for high-dimensional mappings [S1].
*   **Strengths:** Fast optimization, suitable for monocular dynamic scenes, explicit occlusion handling [S1].
*   **Limitations:** Relies on monocular input, which can be challenging for motion/geometry learning compared to multi-view setups (e.g., DeVRF) [S1]. Performance depends on grid resolution trade-offs [S1].
*   **When to use:** When fast training of dynamic scenes from monocular video is required.
*   **When not to use:** When multi-view data is available and higher geometric robustness is needed (consider DeVRF) or when image quality refinement via look-up tables is prioritized over training speed (consider V4D) [S1].

## Background
NDVG was proposed by X. Guo et al. in the paper "Neural Deformable Voxel Grid for Fast Optimization of Dynamic View Synthesis" [S1]. It is based on the canonical scene framework [39,52,36] and voxel-grid optimization for static radiance fields [48,63]. It addresses the challenge of designing a compact deformation model with enough capacity while speeding up training [S1].

## Sources
[S1] Neural Deformable Voxel Grid for Fast Optimization of Dynamic View Synthesis, https://doi.org/10.1007/978-3-031-26319-4_27, fetched 2026-10-26
