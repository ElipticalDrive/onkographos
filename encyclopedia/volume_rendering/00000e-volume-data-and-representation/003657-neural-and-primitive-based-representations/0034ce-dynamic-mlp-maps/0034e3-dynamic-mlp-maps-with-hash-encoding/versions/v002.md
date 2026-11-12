# Dynamic MLP Maps with Hash Encoding


## Overview
A method for representing volumetric videos by predicting MLP parameters via 2D CNNs from axis-aligned orthogonal maps, combined with a multi-level hash encoding for spatial features, allowing real-time rendering of dynamic scenes [S1]. It builds on [Dynamic MLP Maps](7360-dynamic-mlp-maps.md) by combining axis-aligned orthogonal MLP maps predicted by a 2D CNN decoder with multi-level hash encoding for spatial features [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of monocular camera views (video frames) of a dynamic scene [S1].
*   **Output:** Rendered pixel colors $\tilde{C}(\mathbf{r})$ for given camera rays $\mathbf{r}$ [S1].

**Data Structures:**
1.  **MLP Maps:** Three 2D grids (axis-aligned orthogonal planes: XY, YZ, XZ) storing parameters for a shallow MLP [S1]. The parameters are predicted by a shared 2D CNN decoder [S1].
2.  **Hash Tables:** Multi-level hash tables mapping 3D coordinates to high-dimensional feature vectors [S1].
3.  **Occupancy Volume:** A 3D grid of resolution $24 \times 24 \times 48$ per video frame, used for empty space skipping [S1].

**Numbered Steps:**
1.  **Training Phase:**
    1.  For each training ray $\mathbf{r}$, uniformly sample $M=64$ query locations $\mathbf{x}_i$ along the ray within the scene bound [S1].
    2.  For each query point $\mathbf{x}_i$:
        a.  Project $\mathbf{x}_i$ onto the three axis-aligned orthogonal MLP maps to retrieve corresponding MLP parameters $\theta_{xy}, \theta_{yz}, \theta_{xz}$ [S1].
        b.  Embed $\mathbf{x}_i$ into a high-dimensional feature vector $\mathbf{f}_{hash}$ using multi-level hash tables [S1].
        c.  Feed $\mathbf{f}_{hash}$ into the MLP networks defined by parameters $\theta_{xy}, \theta_{yz}, \theta_{xz}$ to predict color $\mathbf{c}_i$ and density $\sigma_i$ [S1].
        d.  Combine predictions from the three MLP maps via element-wise summation to obtain final $\mathbf{c}_i$ and $\sigma_i$ [S1].
    3.  Compute rendered color $\tilde{C}(\mathbf{r})$ using differentiable volume rendering [S1].
    4.  Compute loss $L_c$ as MSE between $\tilde{C}(\mathbf{r})$ and observed color $C(\mathbf{r})$ [S1].
    5.  Compute total loss $L = L_c + \lambda_{KL}L_{KL}$, where $L_{KL}$ is Kullback-Leibler divergence [S1].
    6.  If the scene contains only foreground objects, add mask loss $L_m$ with weight 0.1 [S1].
    7.  Update network weights using the total loss [S1].
2.  **Inference Phase:**
    1.  For each camera ray $\mathbf{r}$, determine the set of occupied voxels using the pre-computed occupancy volume [S1].
    2.  Sample points only within these occupied voxels, with step size $1/256$ of the scene bound diagonal [S1].
    3.  For each sampled point $\mathbf{x}_i$:
        a.  Compute density $\sigma_i$ (and transmittance $T_i$, weight $\omega_i$) to determine visibility [S1].
        b.  If $\omega_i < \tau_2$, ignore the point for color evaluation [S1].
        c.  Otherwise, predict color $\mathbf{c}_i$ and density $\sigma_i$ using the same process as training [S1].
    4.  Compute final pixel color $\tilde{C}(\mathbf{r})$ using volume rendering [S1].

**Formulas:**
*   **Volume Rendering:**
    $$ \tilde{C} (\mathbf{r}) = \sum_{i = 1}^{M} \omega_i \mathbf{c}_i $$
    $$ \omega_i = T_i(1 - \exp(-\sigma_i\delta_i)) $$
    $$ T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j\delta_j\right) $$
    Where $\mathbf{c}_i$ is color, $\sigma_i$ is density, $\delta_i$ is ray step size, $T_i$ is accumulated transmittance, and $\omega_i$ is composition weight [S1].
*   **Color Loss:**
    $$ L_c = \sum_{\mathbf{r} \in \mathcal{R}} || \tilde{C}(\mathbf{r}) - C(\mathbf{r}) ||^2_2 $$
    Where $\mathcal{R}$ is the set of camera rays [S1].
*   **Mask Loss (Optional):**
    $$ L_m = \sum_{\mathbf{r} \in \mathcal{R}} || \tilde{M}(\mathbf{r}) - M(\mathbf{r}) ||^2_2 $$
    Where $\tilde{M}$ is ground truth foreground mask and $M$ is rendered image opacity [S1].
*   **Total Loss:**
    $$ L = L_c + \lambda_{KL}L_{KL} $$
    Where $L_{KL}$ is Kullback-Leibler divergence [S1].

**Parameters:**
*   $\tau_1$: Density threshold for occupancy volume. Value: $5$ [S1].
*   $\tau_2$: Composition weight threshold for filtering points during inference. Value: $1e-3$ [S1].
*   $\lambda_{KL}$: Weight for KL divergence loss. Value: $1e-6$ [S1].
*   Mask loss weight: $0.1$ (if foreground-only scenes) [S1].
*   Training samples per ray: $64$ [S1].
*   Inference sampling step size: $1/256$ of the diagonal distance of the scene bound [S1].

**Edge Cases and Pitfalls:**
*   The method struggles to render very detailed content, such as text and fingers [S1].
*   During inference, points with composition weight $\omega_i < \tau_2$ are explicitly ignored to avoid evaluating color for occluded or transparent regions [S1].

**Empty Space Skipping (ESS):**
*   The occupancy volume is divided into $5 \times 5 \times 5$ subgrids [S1].
*   A volume cell is marked as occupied if any subgrid point has density $> \tau_1$ ($\tau_1 = 5$) [S1].

**Implementation Details:**
*   A custom PyTorch layer for efficient MLP evaluation is implemented based on the MAGMA library, following KiloNeRF [S1].

## Operating characteristics
*   **Speed:** Rendering is 4x faster than the baseline when Empty Space Skipping (ESS) is used [S1].
*   **Storage/Training:** Avoids linear storage and training time increase with frame count, unlike storing per-frame KiloNeRF models (e.g., 300 frames would require 300 KiloNeRF models) [S1].
*   **Limitations:** Struggles to render very detailed content, such as text and fingers [S1].
*   **Hardware:** Implemented using PyTorch and MAGMA library [S1].

## Background
*   **Origin:** Proposed in "Representing Volumetric Videos as Dynamic MLP Maps" [S1].
*   **Development:** Builds on [Dynamic MLP Maps](7360-dynamic-mlp-maps.md) and incorporates techniques from KiloNeRF [S1] and PlenOctrees [S1].
*   **Usage:** Real-time rendering of dynamic volumetric videos from monocular camera input [S1].

## Sources
[S1] Representing Volumetric Videos as Dynamic MLP Maps, https://arxiv.org/abs/2304.06717, fetched 2026-10-29
