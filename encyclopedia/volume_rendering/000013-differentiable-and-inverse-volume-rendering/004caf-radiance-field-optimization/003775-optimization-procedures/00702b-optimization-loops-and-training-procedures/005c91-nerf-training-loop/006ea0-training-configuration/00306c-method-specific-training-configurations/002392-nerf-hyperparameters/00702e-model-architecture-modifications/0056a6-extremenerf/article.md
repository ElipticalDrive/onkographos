# ExtremeNeRF

Also called: Few-shot NeRF under varying-illumination, NeRF Extreme

## Overview
ExtremeNeRF is a few-shot view synthesis framework designed to generate novel views from sparse multi-view images captured under unconstrained, varying illumination [S1]. It solves the problem of illumination-induced geometry distortion in standard NeRF by enforcing multi-view albedo consistency, assuming albedo is an intrinsic, illumination-invariant property of the scene [S1]. Unlike methods requiring massive image collections (e.g., NeRF-W) or those that fail with limited data (e.g., RegNeRF, FreeNeRF), ExtremeNeRF uses a two-stage intrinsic decomposition pipeline: an offline Full-image Intrinsic Decomposition Network (FIDNet) to generate pseudo-albedo ground truths, and an online Patch-wise Intrinsic Decomposition Network (PIDNet) to guide NeRF optimization via albedo consistency losses [S1].

## How it works
ExtremeNeRF builds on the baseline NeRF architecture [Parent](3065-nerf-hyperparameters.md) but augments the training objective with intrinsic decomposition constraints. The method consists of an offline stage to prepare ground truths and an online stage where NeRF and PIDNet are optimized jointly.

### Inputs and Outputs
**Inputs:**
*   A set of $N$ input images $\{I_i\}_{i=1}^N$ with known camera poses $(K_i, T_i)$ and intrinsics $K_i$.
*   $N$ is small (few-shot, e.g., 3 views).

**Outputs:**
*   A trained NeRF MLP $F_\Theta$ mapping $(\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$.
*   A trained PIDNet $F_{PID}$ mapping color patches to intrinsic components.

### Data Structures
1.  **NeRF MLP ($F_\Theta$):** Standard coordinate-based MLP.
    *   Input: 3D position $\mathbf{x} \in \mathbb{R}^3$, viewing direction $\mathbf{d} \in \mathbb{R}^3$.
    *   Output: Volume density $\sigma \in \mathbb{R}$, view-dependent color $\mathbf{c} \in \mathbb{R}^3$.
2.  **FIDNet:** Pre-trained intrinsic decomposition network (based on IIDWW [S1]).
    *   Input: Full input image $I_i$.
    *   Output: Pseudo-albedo map $A_i^{pseudo} \in \mathbb{R}^{H \times W \times 3}$.
3.  **PIDNet ($F_{PID}$):** Patch-wise intrinsic decomposition network.
    *   Input: Color patch $\hat{c}(x)$ of size $S_{patch} \times S_{patch}$.
    *   Output: Extracted albedo $\hat{a}(x)$ and shading $\hat{s}(x)$.

### Numbered Steps

**Stage 1: Offline Pseudo-Albedo Generation**
1.  For each input image $I_i$ in the training set:
    *   Pass $I_i$ through the pre-trained FIDNet.
    *   Store the output as the pseudo-albedo ground truth $A_i^{pseudo}$.
    *   *Note:* FIDNet is not fine-tuned during this stage [S1].

**Stage 2: Joint NeRF and PIDNet Optimization**
For each training iteration:
1.  **Ray Sampling:** Sample rays from the input views and novel view candidates.
2.  **NeRF Rendering:**
    *   For each ray, sample points $\mathbf{x}_t$ along the ray.
    *   Query NeRF $F_\Theta(\mathbf{x}_t, \mathbf{d})$ to get $\sigma_t, \mathbf{c}_t$.
    *   Render color $\hat{c}(x)$ and depth $\hat{d}(x)$ using standard volume rendering equations [G]:
        $$ \hat{c}(x) = \sum_{t=1}^T T_t (1 - \alpha_t) \mathbf{c}_t, \quad \alpha_t = 1 - e^{-\sigma_t \delta_t} $$
        $$ \hat{d}(x) = \sum_{t=1}^T T_t (1 - \alpha_t) t $$
3.  **Geometric Alignment (Pixel Correspondence):**
    *   For a pixel $x$ in the novel view with rendered depth $\hat{d}(x)$, map it to the corresponding pixel $x'$ in an input view $i$:
        $$ \bar{x} = [x^T, 1]^T $$
        $$ \mathbf{x}_w = T_i \hat{d}(x) K_i^{-1} \bar{x} $$
        $$ x' = K_i T_i^{-1} \mathbf{x}_w $$
        where $T_i$ is the camera-to-world transformation and $K_i$ is the intrinsic matrix [S1].
4.  **Projection Error Calculation:**
    *   Calculate the projected depth $d(x')$ from the input view's geometry (or use the input view's depth if available, otherwise use the NeRF's depth at $x'$ projected back). The source defines projection error $E_{proj}$ as:
        $$ E_{proj}(x') = (\hat{d}(x') - \tilde{d}(x'))^2 $$
        where $\hat{d}(x')$ is the rendered depth at the projected location and $\tilde{d}(x')$ is the projected depth from the novel view [S1].
    *   *Pitfall:* Direct minimization of $E_{proj}$ is counterproductive due to occlusions smoothing unrelated surface depths [S1].
5.  **Visibility Mask Weight:**
    *   Compute weight $\omega(x)$ to handle projection errors and occlusions:
        $$ \omega(x) = r_e \left( 1 - \frac{E_{proj}(x')}{M_{proj}} \right) $$
        where $r_e$ is an error rate coefficient that diminishes toward the end of training, and $M_{proj}$ is the maximum projection error [S1].
6.  **PIDNet Intrinsic Decomposition:**
    *   Extract a color patch $\hat{c}(x)$ of size $S_{patch} \times S_{patch}$ centered at $x$ from the rendered novel view.
    *   Pass $\hat{c}(x)$ through PIDNet to get extracted albedo $\hat{a}(x)$ and shading $\hat{s}(x)$ [S1].
7.  **Loss Computation:**
    *   **Color Loss:** Mean squared error between rendered color and ground truth.
        $$ L_{color} = \sum_{x \in S} \|\hat{c}(x) - c_{gt}(x)\|_2^2 $$
    *   **Albedo Consistency Loss ($L_{ac}$):** Enforce that the albedo extracted from the novel view matches the pseudo-albedo from the corresponding input view pixel $x'$.
        $$ L_{ac} = \sum_{x \in P} \omega(x) \|\hat{a}(x) - A_{x'}^{pseudo}\|_2^2 $$
        where $P$ is the set of pixels in the novel view [S1].
    *   **Depth Consistency Loss ($L_{dc}$):** Regularize projection error between adjacent pixels to prevent geometry smoothing errors.
        $$ L_{dc} = \sum_{x' \in P'} \sum_{y \in N(x')} (E_{proj}(y) - E_{proj}(x'))^2 $$
        where $N(x')$ are the 4-neighbor adjacent pixels of $x'$ in the input view [S1].
    *   **Additional Regularization:** Depth smoothness ($L_{ds}$), edge-preserving ($L_{edge}$), intrinsic smoothness ($L_{pid}$), chromaticity consistency ($L_{chrom}$), and frequency regularization [S1].
    *   **Total Loss:**
        $$ L_{total} = L_{color} + \lambda_{ac} L_{ac} + \lambda_{dc} L_{dc} + \dots $$
8.  **Update:** Backpropagate gradients to update NeRF parameters $\Theta$ and PIDNet parameters.

### Formulas and Definitions
*   **Intrinsic Decomposition (Lambertian):** $\log c(x) = \log a(x) + \log s(x)$ [S1].
*   **Intrinsic Decomposition (Real-world):** $\log c(x) = \log a(x) + \log s(x) + l + R$, where $l$ is light color vector and $R$ is non-Lambertian residuals [S1].
*   **Pixel Correspondence:** $x' = (K' T'^{-1} T) \hat{d}(x) K^{-1} \bar{x}$ [S1].
*   **3D Point Projection:** $\mathbf{x}_w = T \hat{d}(x) K^{-1} \bar{x}$ [S1].
*   **Inverse Projection:** $x = K T^{-1} \mathbf{x}_w$ [S1].

### Parameters
*   $S_{patch}$: Patch size for PIDNet input/output. Value: $32 \times 32$ [S1].
*   Training Resolution: Image size. Value: $300 \times 400$ [S1].
*   Training Duration: 70,000 iterations per scene [S1].
*   Hardware: 4 NVIDIA A100 GPUs [S1].
*   $r_e$: Error rate coefficient. Diminishes toward end of training [S1].
*   $M_{proj}$: Maximum projection error. Not given in the sources.

### Edge Cases and Pitfalls
*   **Projection Errors:** Incorrect synthesized depth leads to projection errors. If $x'$ falls outside the field of view or corresponds to an occluded region, $\omega(x)$ should be 0 or low [S1].
*   **Occlusion Handling:** Direct minimization of $E_{proj}$ causes smoothing of unrelated surfaces. $L_{dc}$ and $\omega(x)$ mitigate this [S1].
*   **Geometry Rectification:** As geometry improves, $r_e$ diminishes, reducing the weight of cross-view consistency for pairs with high projection error, allowing occlusions to persist without forcing consistency [S1].
*   **FIDNet Substitution:** Replacing FIDNet with other intrinsic decomposition models (e.g., Das et al. 2022) degrades performance [S1].

## Operating characteristics
*   **Training Time:** 1.5 hours for 3 views on 4 NVIDIA A100 GPUs [S1]. This is significantly faster than most other few-shot NeRF methods (e.g., RegNeRF: 2.4 hours, FreeNeRF: 2 hours) and much faster than methods requiring 10-20 hours [S1].
*   **Memory:** Not given in the sources.
*   **Strengths:**
    *   Works with few-shot inputs (sparse multi-view images) under unconstrained illumination [S1].
    *   Faster training than comparable methods [S1].
    *   FIDNet provides global context-aware guidance, outperforming MLP-based albedo synthesis [S1].
*   **Limitations:**
    *   Relies on the quality of the pre-trained FIDNet (IIDWW).
    *   Performance degrades if FIDNet is replaced with less suitable intrinsic decomposition models [S1].
    *   Requires careful tuning of visibility mask weights and projection error thresholds [S1].
*   **When to use:** Few-shot view synthesis with varying illumination where computational efficiency is important.
*   **When not to use:** When high-quality intrinsic decomposition models are unavailable or when dense multi-view inputs are available (other methods may be more robust).

## Background
ExtremeNeRF was introduced in 2024 by authors from the AAAI conference [S1]. It addresses the limitations of NeRF, which requires constrained illumination and extensive multi-view images, and previous few-shot methods that rely on large datasets or yield sub-optimal results with limited data [S1]. The method builds on the concept of intrinsic image decomposition, separating albedo and shading, and applies it to NeRF optimization via a two-stage pipeline [S1]. It was evaluated on the Phototourism F3 and the newly created NeRF Extreme benchmark, the first in-the-wild benchmark for novel view synthesis under varying illumination [S1].

## Sources
[S1] Few-Shot Neural Radiance Fields under Unconstrained Illumination, https://doi.org/10.1609/aaai.v38i4.28075, fetched 2026-10-27
