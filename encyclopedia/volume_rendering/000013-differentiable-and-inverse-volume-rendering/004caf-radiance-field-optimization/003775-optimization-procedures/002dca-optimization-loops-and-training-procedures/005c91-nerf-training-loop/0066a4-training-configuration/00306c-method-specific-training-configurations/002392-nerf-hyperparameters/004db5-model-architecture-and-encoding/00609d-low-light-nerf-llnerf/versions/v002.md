# Low-Light NeRF (LLNeRF)

## Overview
Low-Light NeRF (LLNeRF) is an unsupervised method that synthesizes normal-light novel views directly from low-dynamic-range (LDR) sRGB images captured in low-light conditions [S1]. It solves the problem of training Neural Radiance Fields on camera-finished sRGB data, where the Image Signal Processor (ISP) introduces non-linearity, low signal-to-noise ratio, and color distortion that standard NeRF cannot handle [S1]. Unlike HDR-NeRF, which requires RAW data with exposure metadata, LLNeRF decomposes the radiance field into view-dependent lighting and view-independent color components, allowing for joint enhancement and reconstruction without ground-truth supervision [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A set of $N$ low-light sRGB images (8-bit/channel, LDR) with known camera intrinsics and extrinsics [S1].
*   **Output:** A radiance field model capable of rendering normal-light, enhanced novel views [S1].

**Data Structures and Network Architecture**
The model replaces the standard NeRF MLP with a decomposed architecture consisting of two branches [S1]:
1.  **View-Dependent Branch ($F_1$):** Predicts a single-channel intensity value $v$. This branch captures lighting-related intensity variations.
    *   Input: Encoded position $\tau$ and view direction $d$.
    *   Output: $v \in \mathbb{R}^1$.
    *   Architecture: As shown in Figure 3(c) of [S1], this involves a Feature MLP (8x256), a Coeff MLP (1x128), and an Enhancement Model $\phi$.
2.  **View-Independent Branch ($F_2$):** Predicts the intrinsic color basis $r$.
    *   Input: Encoded position $\tau$.
    *   Output: $r \in \mathbb{R}^3$ (RGB).
    *   Architecture: As shown in Figure 3(c) of [S1], this involves a Feature MLP (8x256), a Color MLP (1x128), and a Direction MLP (1x128).

The final color $c$ is the element-wise product of these components: $c = v \bullet r$ [S1].

**Numbered Steps**
1.  **Ray Sampling:** For each pixel, generate a ray $r(\lambda) = o_{world} + \lambda d_{world}$ using camera intrinsics and extrinsics [S1]. Sample $K$ points along the ray within the near/far range [S1].
2.  **Positional Encoding:** Apply hierarchical positional encoding $\gamma(\cdot)$ to the 3D position $\mathbf{x}$ and view direction $\mathbf{d}$ [S1]. Let $\tau$ denote the concatenated encoded position features [S1].
3.  **Decomposition Forward Pass:**
    *   Compute the view-dependent intensity: $v_i = F_1(\tau_i, d_i; \Theta_{F1})$ for each sample $i$ [S1].
    *   Compute the view-independent color: $r_i = F_2(\tau_i; \Theta_{F2})$ for each sample $i$ [S1].
    *   Compute the raw color: $c_i = v_i \bullet r_i$ [S1].
4.  **Enhancement:** Apply an enhancement function $\phi$ to the view-dependent component $v$ to produce the enhanced intensity $\hat{v} = \phi(v)$ [S1]. The enhanced color is then $\hat{c}_i = \hat{v}_i \bullet r_i$ [S1]. The function $\phi$ is part of the learnable network parameters, allowing the model to learn the inverse ISP mapping and noise reduction jointly with the scene representation [S1].
5.  **Volume Rendering:** Render the pixel color $\hat{c}_r$ using the standard volume rendering integral with the enhanced colors $\hat{c}$ [S1]:
    $$ \hat{c}_r = \sum_{i=1}^{K} \alpha_i \hat{c}_i $$
    where $\alpha_i = (1 - e^{-\sigma_i \delta_i}) \prod_{j=1}^{i-1} e^{-\sigma_j \delta_j}$, $\sigma_i$ is the volume density predicted by the density branch (standard NeRF density MLP), and $\delta_i = t_{i+1} - t_i$ [S1].
6.  **Optimization:** Minimize the photometric loss between the rendered enhanced image $\hat{c}_r$ and the input low-light image $I$ [S1]. The optimization is unsupervised, relying on the consistency of the decomposition and the enhancement capability of $\phi$ [S1].

**Formulas**
*   Decomposition:
    $$ v = F_1(\tau, d; \Theta_{F1}) $$
    $$ r = F_2(\tau; \Theta_{F2}) $$
    $$ c = v \bullet r $$
    $$ \hat{c} = \phi(v) \bullet r $$
    Where $\bullet$ denotes element-wise multiplication [S1].
*   Rendering:
    $$ \hat{c}_r = \sum_{i} \left( P_{1 - e^{-\sigma_i \delta_i}} e^{-\sum_{j<i} \sigma_j \delta_j} \hat{c}_i \right) $$
    (Note: The source [S1] provides the standard NeRF rendering equation Eq. 2, substituting $c_i$ with $\hat{c}_i$) [S1].

**Parameters**
*   Input images: 8-bit/channel sRGB [S1].
*   Network depths/widths: Specific layer dimensions are provided in Figure 3(c) of [S1] (e.g., Feature MLP 8x256, Coeff MLP 1x128). Not given in the text body.

**Edge Cases and Pitfalls**
*   **Non-linearity:** Standard NeRF assumes linear radiance. sRGB images are non-linear due to ISP gamma correction and tone mapping. LLNeRF handles this by learning the enhancement $\phi$ within the network, effectively inverting the ISP effects [S1].
*   **Noise:** Low-light sRGB images have low signal-to-noise ratio. NeRF is robust to zero-mean noise in RAW data but not to the blended noise in sRGB. The decomposition helps isolate noise into the view-dependent component, which is then suppressed by the enhancement model $\phi$ [S1].
*   **View Inconsistency:** Applying 2D enhancement before NeRF training causes view inconsistency. LLNeRF avoids this by performing enhancement implicitly during 3D optimization [S1].

## Operating characteristics
*   **Time Complexity:** Comparable to standard NeRF training, but requires additional forward passes for the decomposition branches and the enhancement function $\phi$ [S1].
*   **Memory Complexity:** Slightly higher than standard NeRF due to the additional MLPs for $F_1$ and $F_2$ [S1].
*   **Strengths:** Works directly on camera-finished sRGB images without RAW data or exposure metadata; unsupervised; jointly enhances illumination and reconstructs geometry [S1].
*   **Limitations:** Relies on the assumption that lighting variations are primarily view-dependent and can be separated from intrinsic color; performance depends on the capacity of the enhancement model $\phi$ to generalize to in-the-wild scenes [S1].
*   **When to use:** When only low-light sRGB images are available and RAW data is not accessible [S1].
*   **When not to use:** When RAW data with exposure metadata is available (HDR-NeRF may be more appropriate) or when high-dynamic-range scenes are not the primary focus [S1].

## Background
LLNeRF was proposed in "Lighting up NeRF via Unsupervised Decomposition and Enhancement" (ICCV 2023) [S1]. It addresses the limitations of previous methods like HDR-NeRF, which require RAW data, and standard NeRF, which fails on low-light sRGB images due to ISP-induced non-linearity and noise [S1]. The method builds on the observation that lighting intensity is often view-dependent, while intrinsic color is view-independent, allowing for a product decomposition of the radiance field [S1].

## Sources
[S1] Lighting up NeRF via Unsupervised Decomposition and Enhancement, https://doi.org/10.1109/iccv51070.2023.01161, fetched 2026-10-29
