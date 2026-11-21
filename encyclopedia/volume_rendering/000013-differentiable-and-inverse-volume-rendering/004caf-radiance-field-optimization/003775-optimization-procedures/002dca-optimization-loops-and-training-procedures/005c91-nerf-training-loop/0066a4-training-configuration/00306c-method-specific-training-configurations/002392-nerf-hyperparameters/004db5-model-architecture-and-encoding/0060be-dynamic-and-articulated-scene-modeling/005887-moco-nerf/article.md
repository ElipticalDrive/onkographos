# MoCo-NeRF

Also called: Motion-Oriented Compositional Neural Radiance Fields

## Overview
MoCo-NeRF is a framework for free-viewpoint rendering of monocular human videos that models non-rigid body motions as radiance residual fields rather than spatial offsets, using rigid skeletal transformations as a prior [S1]. It separates the canonical radiance representation from pose-dependent non-rigid residuals, allowing direct supervision from ground-truth pixel colors to optimize both components simultaneously [S1]. Unlike sibling methods like A-NeRF which may rely on spatial offsets or surface-based deformations, MoCo-NeRF avoids the high computational complexity and misaligned learning objectives associated with unbounded spatial offset prediction by operating directly in the radiance space [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Monocular video frames $I$, camera poses, and skeletal joint positions $J$ for each frame.
*   **Outputs:** Rendered RGB images $C_{final}$ for arbitrary viewpoints and times.

**Data Structures:**
1.  **Skeletal Weights Volume Decoder:** An explicit volume decoder optimized to store canonical skinning weights $w^c_k$ for $K$ joints [S1].
2.  **Multi-Resolution Hash Encoding (MHE):** A hash grid $\psi_{hash}$ with 16 levels and feature dimension $H=2$ per level [S1]. For multi-subject training, a global MHE $\psi^G_{hash}$ (4 levels, $H=2$) and $N_s$ local MHEs $\{\psi^i_{hash}\}$ are used [S1].
3.  **Learnable Base Code:** A tensor $S \in \mathbb{R}^{64}$ initialized randomly [S1].
4.  **MLP Decoders:**
    *   $M_{rigid}$: Predicts rigid color and density.
    *   $M_{nr}$: Predicts non-rigid color and density residuals.
5.  **Projection Matrices:** $W_p, W_Q, W_K, W_V$ for pose-embedded feature generation [S1].

**Numbered Steps:**

1.  **Canonical Transformation:** Transform an observation point $\mathbf{x}$ at time $t$ to canonical space $\mathbf{x}_c$ using inverse linear blend skinning [S1]:
    $$ \mathbf{x}_c = \sum_{k=1}^K w^o_k (R_k \mathbf{x} + \mathbf{t}_k) $$
    where $R_k, \mathbf{t}_k$ are the rotation and translation of bone $k$, and observation weights $w^o_k$ are computed from canonical weights $w^c_k$ [S1]:
    $$ w^o_k = \frac{w^c_k(R_k \mathbf{x} + \mathbf{t}_k)}{\sum_{j=1}^K w^c_j(R_j \mathbf{x} + \mathbf{t}_j)} $$
    The canonical weights $w^c_k$ are optimized via an explicit volume decoder [S1].

2.  **Rigid Branch Feature Query:** Query multiresolution volumetric features $\mathbf{f}_{\mathbf{x}_c}$ from the MHE $\psi_{hash}$ (or global/local MHEs in multi-subject mode) [S1]:
    $$ \mathbf{f}_{\mathbf{x}_c} = \psi_{hash}(\mathbf{x}_c) $$
    Slice the first half ($H$ dimensions) of $\mathbf{f}_{\mathbf{x}_c}$ at each resolution level to compute $\mathbf{f}^{:H}_{\mathbf{x}_c}$ [S1].

3.  **Rigid Radiance Prediction:** Predict rigid color $\mathbf{c}$ and density $\sigma$ using decoder $M_{rigid}$ [S1]:
    $$ \mathbf{c}, \sigma = M_{rigid}(\mathbf{f}^{:H}_{\mathbf{x}_c}) $$
    *Note: In multi-subject mode, concatenate global features, sliced local features, and subject ID code $\mathbf{S}_i$:*
    $$ \mathbf{c}, \sigma = M_{rigid}(\psi^G_{hash}(\mathbf{x}_c) \oplus (\psi^i_{hash}(\mathbf{x}_c))^{:H} \oplus \mathbf{S}_i) \quad \text{[S1]} $$

4.  **Non-Rigid Feature Preparation:** Use the full feature $\mathbf{f}_{\mathbf{x}_c}$ but freeze the first half representing canonical radiance. Denote the remaining optimized half as $\mathbf{f}^*_{\mathbf{x}_c}$ [S1].

5.  **Pose-Embedded Implicit Feature Generation:**
    a. Initialize learnable base code $\mathbf{S} \in \mathbb{R}^{64}$ [S1].
    b. Apply sinusoidal positional encoding $\gamma$ to 23 joint positions $\mathbf{J} = \{\mathbf{j}_i\}_{i=1}^{23}$ with $L=10$ frequency bands [S1]:
       $$ \gamma(\mathbf{j}_i) = [\mathbf{j}_i, \sin(2^0 \pi \mathbf{j}_i), \cos(2^0 \pi \mathbf{j}_i), \dots, \sin(2^{L-1} \pi \mathbf{j}_i), \cos(2^{L-1} \pi \mathbf{j}_i)] $$
       Resulting in $\gamma(\mathbf{J}) \in \mathbb{R}^{23 \times 63}$ [S1].
    c. Project encoded joints to raw pose-embedded code $\mathbf{P}_{raw}$ using projection matrix $W_p$ [S1]:
       $$ \mathbf{P}_{raw} = W_p \cdot \gamma(\mathbf{J}) $$
    d. Generate query $\mathbf{P}_q$ from $\mathbf{S}$, and key $\mathbf{P}_k$, value $\mathbf{P}_v$ from $\mathbf{P}_{raw}$ via linear projections [S1]:
       $$ \mathbf{P}_q = W_Q \cdot \mathbf{S}, \quad \mathbf{P}_k = W_K \cdot \mathbf{P}_{raw}, \quad \mathbf{P}_v = W_V \cdot \mathbf{P}_{raw} $$
    e. Compute final implicit feature $\mathbf{P}_{nr}$ using cross-attention [S1]:
       $$ \mathbf{P}_{nr} = \text{softmax}(\mathbf{P}_q \cdot \mathbf{P}_k^\top) \cdot \mathbf{P}_v $$

6.  **Non-Rigid Radiance Residual Prediction:** Predict color $\Delta \mathbf{c}$ and density $\Delta \sigma$ residuals using decoder $M_{nr}$ conditioned on $\mathbf{f}^*_{\mathbf{x}_c}$ and $\mathbf{P}_{nr}$ [S1]:
    $$ \Delta \mathbf{c}, \Delta \sigma = M_{nr}(\mathbf{f}^*_{\mathbf{x}_c}; \mathbf{P}_{nr}) $$

7.  **Compositional Rendering:** For each ray $\mathbf{r}$, query $M$ samples. Compute transmittance $T_m$ and final RGB color $C_{final}$ using stratified sampling [S1]:
    $$ T_m = \exp\left(-\sum_{n=1}^{m-1} \sigma_n \delta_n\right) $$
    $$ C_{final}(\mathbf{r}) = \sum_{m=1}^M T_m (1 - \exp(-(\sigma_m + \Delta \sigma_m)\delta_m)) (\mathbf{c}_m + \Delta \mathbf{c}_m) $$
    where $\delta_m$ is the distance between adjacent samples [S1]. Similarly, compute $C_{rigid}$ from $(\mathbf{c}, \sigma)$ for loss calculation [S1].

8.  **Optimization:**
    a. Sample $N_p$ patches per iteration [S1].
    b. Render patches $\hat{P}^r_k$ from $C_{rigid}$ and $\hat{P}^f_k$ from $C_{final}$ [S1].
    c. Compute LPIPS loss using pre-trained VGGNet against ground truth patch $P^{GT}_k$ [S1]:
       $$ \mathcal{L}_{LPIPS} = \text{LPIPS}(\text{VGG}(\hat{P}), \text{VGG}(P^{GT}_k)) $$
    d. Compute MSE losses $\mathcal{L}^r_{MSE}$ and $\mathcal{L}^f_{MSE}$ for rigid and final patches [S1].
    e. Compute SSIM loss $\mathcal{L}^f_{SSIM}$ only for final patches [S1].
    f. Compute total loss $\mathcal{L}^o$ with hyperparameter $\lambda = 0.2$ [S1]:
       $$ \mathcal{L}^r = \mathcal{L}^r_{LPIPS} + \mathcal{L}^r_{MSE} $$
       $$ \mathcal{L}^f = \mathcal{L}^f_{LPIPS} + \mathcal{L}^f_{MSE} + \mathcal{L}^f_{SSIM} $$
       $$ \mathcal{L}^o = \lambda \mathcal{L}^r + (1-\lambda) \mathcal{L}^f $$

**Parameters:**
*   $K$: Total number of joints (23) [S1].
*   $L$: Number of frequency bands in positional encoding (10) [S1].
*   $\lambda$: Loss balancing hyperparameter (0.2) [S1].
*   Optimizer: Adam with betas (0.9, 0.99) [S1].
*   Learning Rate: $5 \times 10^{-5}$ for skeletal weights volume decoder; $5 \times 10^{-3}$ for all other components [S1].
*   MHE Configuration: 16 levels, $H=2$ per level; Global MHE: 4 levels, $H=2$ [S1].
*   Sampling: 6 patches of size $32 \times 32$ per iteration; 256 samples per ray [S1].

**Edge Cases and Pitfalls:**
*   Spatial offset strategies suffer from high computational complexity because a single spatial offset spans an unbounded range and is unique for every 3D point across all input frames and body poses [S1].
*   Spatial offset strategies suffer from misaligned learning objectives because the learning objective of spatial offsets is not directly aligned with conventional RGB-based supervision [S1]. MoCo-NeRF avoids these by modeling residuals directly [S1].

## Operating characteristics
*   **Time Complexity:** Training is significantly faster than offset-based methods. Single-subject training takes 1–2 hours on an RTX 3090 GPU [S1]. Multi-subject training (6 subjects) takes ~2 hours on one RTX 3090 [S1]. This is much faster than HumanNeRF (38.9h single-subject) and NHP (~123h multi-subject) [S1].
*   **Memory Complexity:** Uses a single MHE for single-subject, or global + local MHEs for multi-subject. Feature dimension $H=2$ is small [S1].
*   **Performance:** On ZJU-MoCap single-subject (subjects 377, 386, 387, 392, 393, 394), achieves PSNR 31.06, SSIM 0.9734, LPIPS* 28.83 [S1]. On multi-subject (377, 386, 392), achieves PSNR 31.97, SSIM 0.9781, LPIPS* 24.54 [S1].
*   **Strengths:** Direct color supervision for non-rigid motions; efficient training; supports multi-subject training in a single framework [S1].
*   **Limitations:** Requires skeletal joint positions as input [S1]. Performance depends on the accuracy of the skeletal transformation [S1].

## Background
MoCo-NeRF was proposed by J. Kim et al. in 2024 [S1]. It addresses the limitations of conventional spatial offset modeling in dynamic human rendering, which suffers from high learning complexity and misaligned objectives [S1]. The method builds on the principles of canonical transformation and conditioning described in [Parent](18108-a-nerf.md), specifically utilizing inverse linear blend skinning for rigid transformations [S1]. It is evaluated on the ZJU-MoCap and MonoCap datasets [S1].

## Sources
[S1] Motion-Oriented Compositional Neural Radiance Fields for Monocular Dynamic Human Modeling, https://doi.org/10.1007/978-3-031-72933-1_27, fetched 2026-10-30
