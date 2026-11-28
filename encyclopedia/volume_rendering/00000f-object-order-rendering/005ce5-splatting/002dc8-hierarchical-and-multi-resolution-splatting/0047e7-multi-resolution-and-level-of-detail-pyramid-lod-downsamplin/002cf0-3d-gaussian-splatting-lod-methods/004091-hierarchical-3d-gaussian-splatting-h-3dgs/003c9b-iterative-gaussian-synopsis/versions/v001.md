# Iterative Gaussian Synopsis

Also called: Iterative Gaussian Synopsis Framework, Iterative Gaussian Synopsis Unfolding, Top-Down Unfolding

## Overview
Iterative Gaussian Synopsis is a top-down strategy for constructing a hierarchy of Level of Detail (LOD) representations for 3D Gaussian Splatting (3DGS) [1]. It begins with a full-fidelity model and progressively derives coarser levels through adaptive, learnable pruning of anchors and downsampling of hierarchical grids, ensuring coherent simplification without the error accumulation inherent in bottom-up training [S1]. Unlike bottom-up approaches (e.g., [LapisGS](6143-lapisgs.md)) that build coarse layers first and add enhancements, this method derives coarser levels from the full-fidelity model, ensuring inter-layer consistency [S1]. It differs from independent per-resolution models (e.g., [Scaffold-GS](6143-scaffold-gs.md)) by sharing a foundational codebook and grids, extracting all LODs from a single optimization [S1].

## How it works
**Inputs:** SfM point clouds, training images at four target resolutions.
**Outputs:** A set of LOD representations $\{L_0, \dots, L_3\}$, where $L_3$ is the full-fidelity model and $L_0$ is the coarsest.

**Data Structures:**
1.  **Anchors:** Initialized from SfM point clouds via voxelization. Each anchor $j$ contains:
    *   Position $\mathbf{x}_a \in \mathbb{R}^3$.
    *   Base scale $\mathbf{l}_a \in \mathbb{R}^3$.
    *   $k$ learnable offsets $\mathbf{O}_a \in \mathbb{R}^{k \times 3}$.
    *   Soft-index vector $\mathbf{v}_j \in \mathbb{R}^{N_c}$ for codebook retrieval.
    *   Mask parameters $\mathbf{m}_j \in \mathbb{R}^K$ (where $K$ is the number of Gaussians per anchor).
2.  **Hierarchical Grids:** $L_g = 12$ levels of multiresolution hash grids $\{G_g\}_{g=1}^{L_g}$. Initialized uniformly in $[-10^{-4}, 10^{-4}]$. Feature dimension is 4.
3.  **Anchor Codebook:** Global learnable codebook $\mathbf{C} \in \mathbb{R}^{N_c \times D_c}$ with $N_c=256, D_c=32$. Initialized from $\mathcal{N}(0, 1)$.
4.  **MLPs:**
    *   Fusion MLP $\text{MLP}_{\text{fuse}}$.
    *   Attribute-prediction MLPs $\text{MLP}_{\text{attr}}$ (four MLPs for opacity, color, scale, rotation).
    *   Level-specific variants $\text{MLP}^{(\ell)}_{\text{fuse}}$ and $\text{MLP}^{(\ell)}_{\text{attr}}$ for LOD $\ell$.

**Formulas:**
*   **Hierarchical Feature:** $\mathbf{f}_{\text{hier}}(\mathbf{x}_a) = [\text{Interpolate}(G_g, \mathbf{x}_a)]_{g=1}^{L_g}$, where $\text{Interpolate}$ is multi-linear interpolation [S1].
*   **Local Feature (Training):**
    $$w_j = \text{Softmax}(\mathbf{v}_j)$$
    $$\mathbf{f}_{\text{loc},j} = [\text{Onehot}(\text{argmax}(w_j^T)) + w_j^T - \text{sg}(w_j^T)] \cdot \mathbf{C}$$
    where $\text{sg}(\cdot)$ is stop-gradient [S1].
*   **Local Feature (Inference):** $\text{idx}_j = \text{argmax}(\mathbf{v}_j)$, $\mathbf{f}_{\text{loc},j} = \mathbf{C}[\text{idx}_j]$ [S1].
*   **Fused Feature:** $\mathbf{F}(\mathbf{x}_a) = \text{MLP}_{\text{fuse}}(\mathbf{f}_{\text{hier}}(\mathbf{x}_a) \oplus \mathbf{f}_{\text{loc}})$ [S1].
*   **Gaussian Properties:** $\{\alpha^i, \mathbf{c}^i, \mathbf{s}^i, \mathbf{r}^i\}_{i=0}^k = \mathcal{F}(\mathbf{f}_a, \sigma_c, \vec{d}_c)$, where $\mathcal{F}$ contains $\mathcal{F}_\alpha, \mathcal{F}_c, \mathcal{F}_s, \mathcal{F}_r$ [S1].
*   **Viewing Context:** $\sigma_c = \|\mathbf{x} - \mathbf{x}_c\|_2$, $\vec{d}_c = \frac{\mathbf{x} - \mathbf{x}_c}{\|\mathbf{x} - \mathbf{x}_c\|_2}$, where $\mathbf{x}$ is anchor position and $\mathbf{x}_c$ is camera position [S1].
*   **Gaussian Position:** $\boldsymbol{\mu}_i = \mathbf{x}_a + \mathbf{O}_a^i \cdot \mathbf{l}_a$ [S1].
*   **Mask Computation:**
    $$M_{j,i} = \text{sg}(\mathbf{1}[\sigma(m_{j,i}) > \epsilon] - \sigma(m_{j,i})) + \sigma(m_{j,i})$$
    where $\sigma(\cdot)$ is sigmoid, $\mathbf{1}[\cdot]$ is indicator, $\epsilon$ is threshold [S1].
*   **Pruned Attributes:** $\hat{\alpha}_{j,i} = M_{j,i} \cdot \alpha_{j,i}$, $\hat{s}_{j,i} = M_{j,i} \cdot s_{j,i}$ [S1].
*   **Masking Loss:** $\mathcal{L}_{\text{mask}} = \frac{1}{NK} \sum_j \sum_{i=1}^K \sigma(m_{j,i})$ [S1].
*   **Full-Fidelity Loss:** $\mathcal{L}_{\text{full}} = (1 - \lambda_{\text{SSIM}})\mathcal{L}_1 + \lambda_{\text{SSIM}}\mathcal{L}_{\text{D-SSIM}} + \lambda_{\text{vol}}\mathcal{L}_{\text{vol}}$ [S1].
*   **Volume Regularization:** $\mathcal{L}_{\text{vol}} = \mathbb{E}_i [\prod_{d} s_i^d]$ [S1].

**Parameters:**
*   $N$: Total anchors.
*   $K$: Gaussians per anchor.
*   $\lambda_{\text{SSIM}}$: 0.2 [S1].
*   $\lambda_{\text{vol}}$: Not given in the sources.
*   $\lambda_{\text{group}}$: 0.01 [S1].
*   $T_{\text{full}}$: 30,000 iterations [S1].
*   $T_{\text{unfold}}$: 10,000 iterations per step [S1].
*   Learning rate for masks: 0.01 [S1].
*   Learning rates for LAD MLPs: Fusion 0.005, Opacity 0.002, Color 0.008, Covariance 0.004 [S1].
*   Grid levels per LOD: $L_3$ uses 12, $L_2$ uses 9, $L_1$ uses 6, $L_0$ uses 3 (first $3(\ell+1)$ of 12) [S1].

**Steps:**
1.  **Initialization:** Initialize anchors from SfM points via voxelization. Initialize offsets, scales, MLPs, grids, and codebook [S1].
2.  **Full-Fidelity Training:**
    For $t = 1$ to $T_{\text{full}}$:
    *   Render a sampled view.
    *   Minimize $\mathcal{L}_{\text{full}}$.
    *   Update all parameters [S1].
3.  **Index Finalization:** Set $\text{idx}_j = \text{argmax}(\mathbf{v}_j)$. This model is LOD $L_3$ [S1].
4.  **Iterative Unfolding:**
    For $\ell = 3$ down to 1:
    *   **Restrict Grids:** Use only the coarsest $3\ell$ grid levels for LOD $\ell-1$ [S1].
    *   **Initialize Level-Specific Components:**
        *   Set masks $\mathbf{M} \leftarrow 1$ (all active) [S1].
        *   Copy codebook: $\mathbf{C}^{(\ell-1)} \leftarrow \mathbf{C}$ [S1].
        *   Copy MLPs: $\text{MLP}^{(\ell-1)} \leftarrow \text{MLP}$ [S1].
    *   **Unfold Training:**
        For $t = 1$ to $T_{\text{unfold}}$:
        *   Update masks using straight-through estimator:
            $$M_{j,i} \leftarrow \text{sg}(\mathbf{1}[\sigma(m_{j,i}) > \epsilon] - \sigma(m_{j,i})) + \sigma(m_{j,i})$$
        *   Apply masks to attributes: $\hat{\alpha} \leftarrow \mathbf{M}\alpha$, $\hat{s} \leftarrow \mathbf{M}s$ [S1].
        *   Render at target resolution of LOD $\ell-1$.
        *   Minimize $\mathcal{L}_{\text{LOD}}$ (same structure as $\mathcal{L}_{\text{full}}$) [S1].
        *   Update masks and, depending on configuration, level-specific MLPs and $\mathbf{C}^{(\ell-1)}$ [S1].
        *   Freeze positions, offsets, scales, and grids [S1].
5.  **Return:** $\{L_0, \dots, L_3\}$ [S1].

**Edge Cases and Pitfalls:**
*   **Popping:** The discrete nature of the hierarchy can introduce mild visual "popping" artifacts as new anchors activate during LOD transitions [S1].
*   **Anchor Pruning Condition:** An anchor is inactive if $\sum_{i=1}^K M_{j,i} = 0$ [S1].

## Subtopics

### Level-aware Decoding
A module introducing level-specific sets of fusion and attribute-prediction MLPs ($\text{MLP}^{(\ell)}_{\text{fuse}}$ and $\text{MLP}^{(\ell)}_{\text{attr}}$) fine-tuned specifically for LOD $\ell$ to interpret modulated features optimally [S1]. It applies these MLPs to downsampled hierarchical grids and modulated local features [S1]. Learning rates: Fusion 0.005, Opacity 0.002, Color 0.008, Covariance 0.004 [S1].

### Coherent Basis Modulation
A module that fuses level-adapted local features retrieved from a modulated codebook with downsampled hierarchical features, decoded using frozen, foundational MLPs established during initial full-fidelity training [S1]. The codebook for each LOD is copied from the foundational codebook and fine-tuned, preserving coherence of the shared basis [S1].

## Operating characteristics
*   **Complexity:** Introduces additional training complexity compared to baseline methods [S1].
*   **Performance:** Evaluated on PSNR, SSIM, and LPIPS. The Full Model consistently achieves the best performance [S1].
*   **Storage:** Requires substantially less storage than independent per-resolution models, enabling lightweight upgrades between levels [S1].
*   **Strengths:** Coherent simplification, improved inter-layer consistency, compactness [S1].
*   **Limitations:** Limited to static scenes, potential for mild visual "popping" artifacts [S1].
*   **Usage:** Designed for progressive streaming, real-time 3DGS deployment, and balancing compactness with high-fidelity rendering [S1].

## Background
Originated in the paper "Unfolding 3D Gaussian Splatting via Iterative Gaussian Synopsis" [S1]. It addresses challenges in deploying large-scale 3DGS models for progressive and adaptive rendering by overcoming error accumulation and optimization limitations of bottom-up approaches like LapisGS [S1].

## Sources
[S1] Unfolding 3D Gaussian Splatting via Iterative Gaussian Synopsis., https://doi.org/10.1109/tvcg.2026.3746672, fetched 2026-10-26
