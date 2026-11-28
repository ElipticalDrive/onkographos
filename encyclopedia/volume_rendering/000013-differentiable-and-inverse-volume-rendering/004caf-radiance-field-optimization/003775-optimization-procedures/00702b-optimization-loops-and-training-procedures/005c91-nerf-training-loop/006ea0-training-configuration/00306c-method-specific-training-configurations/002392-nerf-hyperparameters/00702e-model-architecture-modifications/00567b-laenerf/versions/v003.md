# LAENeRF

Also called: Local Appearance Editing for Neural Radiance Fields

## Overview
LAENeRF is a unified framework for the photorealistic and non-photorealistic local appearance editing of pre-trained Neural Radiance Fields (NeRFs) [S1]. It solves the problem of applying recoloring or stylization to arbitrary 3D regions while minimizing background artifacts and maintaining interactivity. Unlike methods that rely on semantic segmentation or global style transfer, LAENeRF uses a 3D edit grid for spatial selection and learns a palette-based color decomposition mapped from estimated ray terminations [S1]. It is distinguished by its ability to model smooth transitions between edited and unedited regions using a growing queue mechanism and its efficient distillation process that transfers edits back to the original radiance field parameters [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A pre-trained NeRF model $\Theta_{NeRF}$ (parameterized by MLPs predicting density $\sigma$ and color $\mathbf{c}$), a set of training rays $R$, and a user-defined selection mask $E$ (the edit grid).
*   **Output:** An edited radiance field where the region defined by $E$ is recolored or stylized, with smooth transitions to the surrounding geometry.

**Data Structures:**
1.  **Edit Grid ($E$):** A 3D voxel grid of resolution $128^3$ [S1]. It acts as a bitfield where bits are set to 1 for voxels containing selected content. It is a subset of the occupancy grid $O$ from Instant-NGP [S1].
2.  **Growing Queue ($G$):** A queue storing direct neighbors of voxels in $E$ to facilitate region growing and smooth transition modeling [S1].
3.  **Growing Grid ($\mathcal{G}$):** A grid constructed from the final state of $G$, used to compute transition weights [S1].
4.  **LAENeRF Module Parameters ($\Theta_{LAENeRF}$):**
    *   **Color Palette ($\hat{P}$):** A learnable tensor of shape $\mathbb{R}^{N_P \times 3}$, where $N_P$ is the number of base colors (initialized to 8) [S1].
    *   **Hash Grid Encoder:** A trainable multi-resolution hash grid (from Instant-NGP) encoding spatial positions [S1].
    *   **MLPs:** Shallow networks taking encoded positions and view directions to predict weights $\hat{w}$ and offsets $\hat{\delta}$ [S1].

**Numbered Steps:**

1.  **Region Selection:**
    The user scribbles on the 2D projection of the scene. For each selected pixel, a ray is cast through the pre-trained NeRF. The estimated ray termination $x_{term}$ is computed (see Formula 1). Each $x_{term}$ is mapped to the nearest voxel in $E$, setting the corresponding bit in the underlying bitfield to 1 [S1].

2.  **Region Growing:**
    To enable smooth transitions, a growing queue $G$ stores direct neighbors of voxels in $E$. During region growing, the current voxel is added to $E$ and its neighbors to $G$ if the occupancy grid $O$ is set ($O=1$) [S1]. This process continues until the user stops or a condition is met, creating a boundary zone for blending [S1].

3.  **Featurization and Color Prediction:**
    For rays intersecting $E$, the estimated ray termination $x_{term}$ is featurized using the trainable multi-resolution hash grid to produce $\xi(x_{term})$. This is passed to shallow MLPs along with a spherical harmonics encoding of the viewing direction $d$ to predict:
    *   Per-ray barycentric weights $\hat{w} \in [0, 1]^{N_P}$ [S1].
    *   View-dependent offsets $\hat{\delta} \in [-1, 1]^3$ [S1].
    
    The predicted color $\hat{c}$ is composed using the learnable palette $\hat{P}$:
    $$ \hat{c} = \text{clamp}(\hat{w}^T \hat{P} + \hat{\delta}) $$
    where $\text{clamp}(\cdot)$ restricts values to $[0, 1]$ [S1].

4.  **Optimization:**
    The LAENeRF module is optimized by minimizing a composite loss function over a set of rays $R$. The pipeline is fully differentiable [S1]. The total loss includes:
    *   **Content Loss:** $L_{content} = \|\hat{c} - \mathcal{C}(r)\|_2^2$, where $\mathcal{C}(r)$ is the ground truth color [S1].
    *   **Style Loss (Optional):** $L_{style} = \lambda_{style} \|G(s) - G(\hat{c})\|_2^2$, where $s$ is a style image and $G(\cdot)$ is the Gram matrix of features from a semantic encoder (e.g., VGG19 conv5, conv6, conv7) [S1].
    *   **Total Variation Loss:** $L_{TV} = \lambda_{TV} \|\nabla \hat{c} \cdot (1 - \text{depth\_guide})\|_2^2$, restricting smoothing to regions without depth discontinuities [S1].
    *   **Depth Discontinuity Loss:** $L_{depth-disc} = -\lambda_{depth-disc} \|\nabla \hat{c} \cdot \text{depth\_guide}\|_2^2$, encouraging gradients at depth edges [S1].
    *   **Offset Loss:** $L_{offset} = \lambda_{offset} \|\hat{\delta}\|_2^2$, preventing extreme high-frequency offsets [S1].
    *   **Palette Regularization:** $L_{palette} = \|\lfloor \hat{P} \rfloor \cdot \hat{P}\|_2^2$, ensuring valid RGB colors [S1].
    *   **Weight Loss:** $L_{weight} = \lambda_{weight} (1 - \|\hat{w}\|_\infty)$, encouraging sparse weight usage [S1].
    *   **Smooth Transition Loss (Optional):** $L_{smooth} = \lambda_{smooth} \|(\hat{c} - c)^2 \cdot d_{trans}\|_1$, applied when $d_{trans} \in (0, 1]$ [S1].

5.  **Distillation/Transfer:**
    After training $\Theta_{LAENeRF}$, changes are transferred to $\Theta_{NeRF}$ via distillation. A modified training dataset is created where the target color is $T_\alpha \hat{c} + (1-T_\alpha)\mathcal{C}(r)$. Rays not intersecting $E$ use the ground truth color $\mathcal{C}(r)$. The pre-trained NeRF is fine-tuned using this dataset for $7 \times 10^4$ iterations [S1].

**Formulas:**

1.  **Estimated Depth:**
    $$ \zeta = \sum_{i=1}^{N} T_i (1 - \exp(-\sigma_i \delta_i)) t_{i+1} $$
    where $T_i$ is transmittance, $\sigma_i$ is density, $\delta_i = t_{i+1} - t_i$, and $t_{i+1}$ is the next sample position [S1].

2.  **Ray Termination:**
    $$ x_{term} = o + \zeta d $$
    where $o$ is the camera position and $d$ is the viewing direction [S1].

3.  **Accumulated Alpha in Edit Grid:**
    $$ T_\alpha = \sum_{i=1}^{N} \mathbbm{1}(o + t_i d \in E) T_i (1 - \exp(-\sigma_i \delta_i)) $$
    where $\mathbbm{1}(\cdot)$ is the indicator function. Rays are only processed if $T_\alpha > \tau_{edit}$ (default 0.5) [S1].

4.  **Transition Weight:**
    $$ d_{trans} = 1 - \frac{\min(d_{min}, \tau_{dist})}{\tau_{dist}} $$
    where $d_{min}$ is the minimum distance from $x_{term}$ to the growing grid $\mathcal{G}$, and $\tau_{dist}$ controls transition size (default $1 \times 10^{-2}$) [S1].

5.  **Depth Guide:**
    $$ (\text{depth\_guide})_{i,j} = \left( |\text{depth}_{i,j+1} - \text{depth}_{i,j}|, |\text{depth}_{i+1,j} - \text{depth}_{i,j}| \right) $$
    Used to mask Total Variation and Depth Discontinuity losses [S1].

**Parameters:**
*   **Edit Grid Resolution:** $128^3$ [S1].
*   **Initial Palette Size ($N_P$):** 8 [S1].
*   **Edit Threshold ($\tau_{edit}$):** 0.5 [S1].
*   **Transition Size ($\tau_{dist}$):** $1 \times 10^{-2}$ [S1].
*   **Learning Rate:** $1 \times 10^{-3}$ for all components except $\hat{P}$ ($1 \times 10^{-2}$) [S1].
*   **LAENeRF Training Iterations:** $1 \times 10^5$ [S1].
*   **Distillation Iterations:** $7 \times 10^4$ [S1].
*   **Style Weight ($\lambda_{style}$):** $1.3 \times 10^2$ for stylization, 0 for recoloring [S1].
*   **TV Weight ($\lambda_{TV}$):** $1 \times 10^{-4}$ [S1].
*   **Depth Disc Weight ($\lambda_{depth-disc}$):** $5 \times 10^{-4}$ [S1].
*   **Weight Loss ($\lambda_{weight}$):** $1 \times 10^{-7}$ [S1].
*   **Offset Loss ($\lambda_{offset}$):** $5 \times 10^{-5}$ [S1].
*   **Smooth Loss ($\lambda_{smooth}$):** $1 \times 10^{-3}$ [S1].

**Pitfalls:**
*   **Small Structure Loss:** Naive stylization using only $L_{content}$ and $L_{style}$ eliminates small structures [S1].
*   **Geometry Noise:** Imperfect geometry in the pre-trained NeRF causes noise in $x_{term}$, requiring regularization ($L_{TV}, L_{offset}$) [S1].
*   **Boundary Discontinuities:** Sharp color changes occur at the boundary of $E$ if adjacent cells are occupied; mitigated by the growing queue and $L_{smooth}$ [S1].
*   **Palette Pruning:** Color palettes that do not contribute significantly should be removed after $1.5 \times 10^3$ iterations [S1].

## Subtopics

### Geometry-Aware Stylization with Depth Estimation
A stylization technique that respects the learned geometry of the pre-trained NeRF by performing depth estimation to counteract the removal of small structures common in naive 2D style transfer. It achieves this by training LAENeRF without $L_{style}$, $L_{offset}$, and $L_{TV}$ for the first 1000 iterations to achieve a well-initialized palette $\hat{P}$ before applying geometry-aware losses [S1].

## Operating characteristics
*   **Time Complexity:** The method is interactive, with previews available after $\sim 20$ seconds of LAENeRF optimization [S1]. Distillation takes no longer than 5 minutes on an NVIDIA RTX 4090 [S1].
*   **Memory Complexity:** Uses a lightweight NeRF-like network and a $128^3$ edit grid, significantly reducing memory requirements compared to full scene re-optimization [S1].
*   **Performance:** LAENeRF reduces background MSE error rates by 59% compared to PaletteNeRF with semantic guidance on the LLFF dataset [S1]. It introduces fewer background artifacts than Ref-NPR and is the only compared method capable of modeling smooth transitions between original and edited content [S1].
*   **Strengths:** Interactive local editing, smooth transitions, low background artifacts, unified recoloring/stylization.
*   **Limitations:** Relies on the quality of the pre-trained NeRF's geometry; imperfect geometry leads to noise in ray terminations [S1].

## Background
LAENeRF was proposed by Lukas Radl, Michael Steiner, Andreas Kurz, and Markus Steinberger from Graz University of Technology [S1]. It was published in CVPR 2024 (arXiv:2312.09913v2, March 25, 2024) [S1]. The method builds on NeRFShop [S1] for region selection and Instant-NGP [S1] for the hash grid encoder, implementing on the torch-ngp framework [S1]. It is evaluated on NeRF-Synthetic, LLFF, and mip-NeRF 360 datasets [S1].

## Sources
[S1] LAENeRF: Local Appearance Editing for Neural Radiance Fields, https://doi.org/10.1109/cvpr52733.2024.00475, fetched 2026-10-26
[S2] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
