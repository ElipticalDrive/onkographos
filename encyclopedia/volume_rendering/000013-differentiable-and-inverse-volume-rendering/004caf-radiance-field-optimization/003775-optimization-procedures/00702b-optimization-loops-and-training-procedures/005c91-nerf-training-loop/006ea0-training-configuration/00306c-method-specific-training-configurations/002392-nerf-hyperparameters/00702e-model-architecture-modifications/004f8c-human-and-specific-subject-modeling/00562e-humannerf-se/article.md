# HumanNeRF-SE

Also called: HumanNeRF with Sparse Encoding

## Overview
HumanNeRF-SE is a method for synthesizing novel pose images of humans from monocular video inputs by combining explicit SMPL shape priors with an implicit NeRF representation [S1]. It solves the problems of overfitting and high computational complexity inherent in previous monocular human NeRF methods [S1]. Unlike sibling methods that rely on frame-level features or learnable blend weights, HumanNeRF-SE uses frozen SMPL blending weights for rigid deformation and point-level spatial-aware features for non-rigid refinement, ensuring pose generalization without pose leakage [S1].

## How it works
**Inputs:** Monocular video frames $\{I_n\}$, corresponding SMPL mesh vertices $v$ and pose parameters $J$ (rotation/translation of joints relative to T-pose) [S1].
**Outputs:** Rendered image color $C(\mathbf{r})$ for a ray $\mathbf{r}$ [S1].

**Data Structures:**
*   **Voxel Volume $V$:** A sparse 3D grid. Each voxel containing SMPL vertices stores a tuple $(n_v, w_1, \dots, w_{24})$, where $n_v$ is the vertex count (occupancy) and $w_j$ are the 24 Linear Blending Skinning (LBS) weights from SMPL [S1]. Empty voxels store $(0, 0, \dots, 0)$ [S1].
*   **Spatial-Aware Features $F_s$:** The result of channel-by-channel convolution on $V$ [S1].

**Algorithm Steps:**

1.  **Voxelization:**
    Calculate the bounding box of SMPL vertices $v$ (min/max coordinates). Scale the bounding box to a set voxel size $v_s$. Find the least common multiple of the axes dimensions and 32. Scale vertices down by $v_s$ after subtracting the minimum. Construct the voxel volume $V$ using Sparse Convolution $Sp$:
    $$ V = Sp(v, v_s, W) $$
    [S1]

2.  **Conv-Filter (Feature Extraction & Point Filtering):**
    Initialize a convolution kernel to one. Perform channel-by-channel convolution on $V$ to obtain spatially-aware features $F_s$. For a voxel at position $(h, w, d)$ with kernel size $k$:
    $$ F_{s,i} = \sum_{m=h-k}^{h'-k} \sum_{n=w-k}^{w'-k} \sum_{t=d-k}^{d'-k} \nu_{m,n,t,i} \cdot \vartheta_{m,n,t,i} $$
    where $\nu$ is the voxel value and $\vartheta$ is the kernel weight [S1].
    Query the occupancy of $F_s$ at sampling point $\mathbf{x}$:
    $$ \text{if } F_s(\mathbf{x}) = 0: \quad \mathbf{x}_o \text{ (filtered out)}, \sigma(\mathbf{x}_o) = 0 $$
    $$ \text{if } F_s(\mathbf{x}) > 0: \quad \mathbf{x}_r \text{ (kept, human-related)} $$
    [S1]

3.  **Point Canonicalization (Deformation):**
    *   **Rigid Deformation ($D$):** Query the nearest neighbor SMPL weights $W_{smpl} = K(\mathbf{x}_r, v)$. Compute rigid deformation $\mathbf{x}_{rcnl}$ using the frozen SMPL blending weights $w_j$ and joint transformations $(R_j, T_j)$ relative to the T-pose:
        $$ \mathbf{x}_{rcnl} = \sum_{j \in J} w_j ((\mathbf{x}_r \cdot R_j) + T_j) $$
        where $J$ is the set of joints [S1].
    *   **Non-Rigid Refinement ($P$):** Refine the coarse coordinates using a network $P$ conditioned on the point-level spatial-aware features $F_s$ and the kept points $\mathbf{x}_r$:
        $$ \mathbf{x}_{cnl} = \mathbf{x}_{rcnl} + P(F_s, \mathbf{x}_r) $$
        [S1]

4.  **Appearance Net:**
    Encode $\mathbf{x}_{cnl}$ using positional encoding (standard NeRF). Pass through an MLP $M_\sigma$ to output color $\mathbf{c}$ and density $\sigma$:
    $$ \mathbf{c}, \sigma = M_\sigma(\mathbf{x}_{cnl}) $$
    [S1]

5.  **Rendering:**
    Render the image using differentiable volume rendering. For ray $\mathbf{r}$ with $N_s$ samples:
    $$ \tilde{C}_t(\mathbf{r}) = \sum_{k=1}^{N_s} \alpha_i \mathbf{c}(\mathbf{x}_i) \prod_{j=1}^{i-1} (1 - \alpha_j) $$
    where $\alpha_i = 1 - \exp(-\sigma(\mathbf{x}_i) \Delta t_i)$ [S1].

**Loss Function:**
$$ L = L_{LPIPS} + \lambda L_{MSE} $$
$$ L_{MSE} = \frac{1}{\|\mathbf{R}\|} \sum_{\mathbf{r} \in \mathbf{R}} \|\tilde{C}(\mathbf{r}) - C(\mathbf{r})\|^2 $$
$$ L_{LPIPS} = l_{pips}(\Theta_{fore}(\tilde{I}_i), \Theta_{fore}(I_i)) $$
where $\Theta_{fore}$ is the foreground mask derived from density, and $\lambda$ is the MSE weight [S1].

**Parameters:**
*   $\lambda$: Weight for MSE loss term [S1].
*   $v_s$: Voxel size for bounding box scaling [S1].
*   Rigid deformation uses **frozen** SMPL blending weights (no fine-tuning) [S1].
*   Non-rigid deformation is conditioned on **point-level** features $F_s$, not frame-level features [S1].

**Pitfalls:**
*   Using nearest SMPL vertex blending weights directly fills the space with incorrect colors because training deforms irrelevant points onto the body [S1].
*   Learning blending weights from scratch or fine-tuning SMPL weights leads to overfitting on restricted poses [S1].
*   Direct nearest weight rigid deformation ignores non-rigid clothing deformation; solved by adding $P(F_s, \mathbf{x}_r)$ [S1].
*   Pose leakage in datasets like ZJU-MOCAP due to action repetition; mitigated by few-shot training and diverse evaluation videos [S1].

## Operating characteristics
*   **Complexity:** Reduces required sampling points from ~300K to <100K [S1]. Uses <1% of learnable parameters compared to previous methods [S1].
*   **Performance:** Reduces training time by 95% (1/20 of previous methods) [S1]. Increases rendering speed by 15x without acceleration modules [S1].
*   **Metrics (ZJU-MOCAP Few-Shot):** PSNR=31.09, SSIM=0.974, LPIPS=24.085 [S1]. Outperforms HumanNeRF (PSNR 30.10) and MonoHuman (PSNR 30.11) [S1].
*   **Strengths:** Generalizes to novel poses without overfitting; works with monocular input; fast training/inference [S1].
*   **Limitations:** Relies on SMPL accuracy; performance depends on the quality of the initial SMPL fit [S1].

## Background
HumanNeRF-SE was introduced in 2024 by Miles et al. [S1]. It addresses the limitations of prior monocular human NeRF methods that required large numbers of optimizable parameters and multi-camera data [S1]. It builds on the SMPL model [29] and Sparse Convolution [7] techniques [S1].

## Sources
[S1] HumanNeRF-SE: A Simple yet Effective Approach to Animate HumanNeRF with Diverse Poses, https://doi.org/10.1109/cvpr52733.2024.00145, fetched 2026-10-30
