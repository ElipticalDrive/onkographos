# LAENeRF

Also called: Local Appearance Editing for Neural Radiance Fields

## Overview
LAENeRF is a unified framework for the photorealistic and non-photorealistic local appearance editing of pre-trained Neural Radiance Fields (NeRFs) [S1]. It solves the problem of recoloring or stylizing arbitrary 3D regions while minimizing background artifacts and maintaining interactivity [S1]. Unlike methods that rely on semantic segmentation masks or global style transfer, LAENeRF uses a 3D edit grid for content selection and a palette-based neural module to learn a mapping from estimated ray terminations to output colors [S1]. It is distinguished by its ability to model smooth transitions between the edited region and the original scene content using a growing queue mechanism, and by its efficient distillation step that transfers edits back to the original radiance field parameters [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A pre-trained NeRF model $\Theta_{NeRF}$ (typically Instant-NGP based), a set of training rays $R$, a user-defined 2D mask on a reference view, and optionally a style image $s$ for stylization.
*   **Output:** An edited NeRF model $\Theta_{NeRF}'$ and/or a set of learned palette parameters $\hat{P}$ for interactive recoloring.

**Data Structures:**
1.  **Edit Grid ($E$):** A 3D voxel grid of resolution $128^3$ (matching Instant-NGP's occupancy grid resolution) [S1]. It is a bitfield where bits are set to 1 for voxels belonging to the selected region.
2.  **Growing Queue ($G$):** A queue storing direct neighbors of voxels in $E$ to facilitate region growing and smooth transitions [S1].
3.  **Growing Grid ($\mathcal{G}$):** A 3D grid constructed from the growing queue $G$, used to compute transition weights [S1].
4.  **LAENeRF Module Parameters ($\Theta_{LAENeRF}$):**
    *   **Color Palette ($\hat{P}$):** A learnable tensor of shape $N_{\hat{P}} \times 3$, where $N_{\hat{P}}$ is the number of base colors (initialized to 8) [S1].
    *   **Hash Grid:** A trainable multi-resolution hash grid (from Instant-NGP) for featurizing input positions [S1].
    *   **MLPs:** Shallow multi-layer perceptrons that map featurized positions to weights and offsets [S1].

**Numbered Steps:**

1.  **Region Selection:**
    The user scribbles on a 2D projection of the scene. For each selected pixel, a ray is cast through the pre-trained NeRF $\Theta_{NeRF}$ to compute the estimated ray termination $x_{term}$ [S1].
    *   Compute estimated depth $\zeta$ along the ray:
        $$ \zeta = \sum_{i=1}^{N} T_i (1 - \exp(-\sigma_i \delta_i)) t_{i+1} $$
        where $T_i$ is transmittance, $\sigma_i$ is density, $\delta_i = t_{i+1} - t_i$, and $t_{i+1}$ is the next sample position [S1].
    *   Compute termination point:
        $$ x_{term} = o + \zeta d $$
        where $o$ is the camera position and $d$ is the viewing direction [S1].
    *   Map $x_{term}$ to the nearest voxel in the edit grid $E$ and set the corresponding bit to 1 [S1].

2.  **Region Growing:**
    To ensure smooth transitions, a growing queue $G$ is initialized with neighbors of the selected voxels [S1].
    *   While $G$ is not empty:
        *   Pop a voxel $v$ from $G$.
        *   If the occupancy grid $O$ at $v$ is set ($O=1$):
            *   Add $v$ to $E$.
            *   Add direct neighbors of $v$ to $G$ [S1].
    *   Construct the growing grid $\mathcal{G}$ from the final state of $G$ [S1].

3.  **Featurization and Prediction:**
    For each ray $r \in R$ intersecting $E$:
    *   Check if the accumulated alpha inside the edit grid $T_\alpha$ exceeds the edit threshold $\tau_{edit} = 0.5$ to handle occlusions and resolution errors [S1]:
        $$ T_\alpha = \sum_{i=1}^{N} \mathbbm{1}(o + t_i d \in E) T_i (1 - \exp(-\sigma_i \delta_i)) $$
    *   If $T_\alpha > \tau_{edit}$:
        *   Featurize $x_{term}$ using the trainable hash grid to get $\xi(x_{term})$ [S1].
        *   Featurize the viewing direction $d$ using spherical harmonics encoding [S1].
        *   Pass $\xi(x_{term})$ and encoded $d$ through shallow MLPs to predict:
            *   Per-ray barycentric weights $\hat{w} \in [0, 1]^{N_{\hat{P}}}$ [S1].
            *   View-dependent offsets $\hat{\delta} \in [-1, 1]^3$ [S1].
        *   Compose the predicted color $\hat{c}$:
            $$ \hat{c} = \text{clamp}(\hat{w}^T \hat{P} + \hat{\delta}) $$
            where $\text{clamp}$ restricts values to $[0, 1]$ [S1].

4.  **Optimization:**
    Optimize $\Theta_{LAENeRF}$ by minimizing the total loss $\mathcal{L}$ over the set of rays $R$ [S1]:
    $$ \mathcal{L} = \mathcal{L}_{content} + \mathcal{L}_{style} + \mathcal{L}_{TV} + \mathcal{L}_{depth-disc} + \mathcal{L}_{weight} + \mathcal{L}_{offset} + \mathcal{L}_{palette} + \mathcal{L}_{smooth} $$
    
    *   **Content Loss:**
        $$ \mathcal{L}_{content} = \|\hat{c} - \mathcal{C}(r)\|_2^2 $$
        where $\mathcal{C}(r)$ is the ground truth color [S1].
    *   **Style Loss (optional):**
        $$ \mathcal{L}_{style} = \lambda_{style} \|G(s) - G(\hat{c})\|_2^2 $$
        where $G(\cdot)$ is the Gram matrix of features from a semantic encoder (e.g., VGG19 conv5, conv6, conv7) [S1].
    *   **Total Variation Loss (geometry-aware):**
        $$ \mathcal{L}_{TV} = \lambda_{TV} \|\nabla \hat{c} \cdot (1 - \text{depth\_guide})\|_2^2 $$
        where $\nabla \hat{c}$ denotes image gradients in x/y directions, and $\text{depth\_guide}_{i,j} = (|\text{depth}_{i,j+1} - \text{depth}_{i,j}|, |\text{depth}_{i+1,j} - \text{depth}_{i,j}|)$ restricts the loss to regions without depth discontinuities [S1].
    *   **Depth Discontinuity Loss:**
        $$ \mathcal{L}_{depth-disc} = -\lambda_{depth-disc} \|\nabla \hat{c} \cdot \text{depth\_guide}\|_2^2 $$
        This encourages gradients to be placed in regions of depth discontinuities [S1].
    *   **Weight Loss (sparsity):**
        $$ \mathcal{L}_{weight} = \lambda_{weight} (1 - \|\hat{w}\|_\infty) $$
        Encourages sparse per-pixel predictions [S1].
    *   **Offset Loss:**
        $$ \mathcal{L}_{offset} = \lambda_{offset} \|\hat{\delta}\|_2^2 $$
        Prevents extreme high-frequency offset solutions [S1].
    *   **Palette Regularization:**
        $$ \mathcal{L}_{palette} = \|\lfloor \hat{P} \rfloor \cdot \hat{P}\|_2^2 $$
        Guarantees valid colors in RGB space ($\hat{P}_{i,j} \in [0, 1]$) [S1].
    *   **Smooth Transition Loss (optional):**
        $$ \mathcal{L}_{smooth} = \lambda_{smooth} \|(\hat{c} - c)^2 \cdot d_{trans}\|_1 $$
        Adds to content loss when $d_{trans} \in (0, 1]$ for stylization [S1].

5.  **Distillation/Transfer:**
    After training $\Theta_{LAENeRF}$, transfer changes to $\Theta_{NeRF}$ via fine-tuning [S1].
    *   Compute per-ray transition weights $d_{trans}$ for smooth blending at boundaries [S1]:
        *   Raymarch through $\mathcal{G}$ and $O$ to obtain estimated ray terminations for rays intersecting $\mathcal{G}$, resulting in point cloud $\mathcal{Z}$ [S1].
        *   For each $x_{term}$ from $E$, compute minimum distance to $\mathcal{Z}$:
            $$ d_{min} = \min_{y \in \mathcal{Z}} \|x_{term} - y\|_2 $$
        *   Compute transition weight:
            $$ d_{trans} = 1 - \frac{\min(d_{min}, \tau_{dist})}{\tau_{dist}} $$
            where $\tau_{dist}$ controls transition size (typically $1 \times 10^{-2}$) [S1].
    *   Construct modified training dataset:
        *   For rays intersecting $E$: Target color $C_{target} = T_\alpha \hat{c} + (1 - T_\alpha) \mathcal{C}(r)$ [S1].
        *   For rays not intersecting $E$: Target color $C_{target} = \mathcal{C}(r)$ [S1].
    *   Fine-tune $\Theta_{NeRF}$ using the modified dataset for $7 \times 10^4$ iterations [S1].
    *   For interactive recoloring, interpolate the learned palette $\hat{P}$ and a user-modified palette $\hat{P}_{mod}$ based on $d_{trans}$ [S1].

**Parameters and Typical Values:**
*   Edit grid resolution: $128^3$ [S1].
*   Edit threshold $\tau_{edit}$: 0.5 [S1].
*   Initial palette size $N_{\hat{P}}$: 8 [S1].
*   Learning rate: $1 \times 10^{-3}$ for all components, except $\hat{P}$ which is $1 \times 10^{-2}$ [S1].
*   Training iterations for LAENeRF: $1 \times 10^5$ [S1].
*   Distillation iterations: $7 \times 10^4$ [S1].
*   Style weight $\lambda_{style}$: $1.3 \times 10^2$ for stylization, 0 for recoloring [S1].
*   $\lambda_{TV}$: $1 \times 10^{-4}$ [S1].
*   $\lambda_{depth-disc}$: $5 \times 10^{-4}$ [S1].
*   $\lambda_{weight}$: $1 \times 10^{-7}$ [S1].
*   $\lambda_{offset}$: $5 \times 10^{-5}$ [S1].
*   $\lambda_{smooth}$: $1 \times 10^{-3}$ [S1].
*   $\tau_{dist}$: $1 \times 10^{-2}$ (scene-dependent) [S1].

**Edge Cases and Pitfalls:**
*   **Small Structures:** Naive stylization using only $\mathcal{L}_{content}$ and $\mathcal{L}_{style}$ often eliminates small structures [S1]. Geometry-aware losses ($\mathcal{L}_{TV}, \mathcal{L}_{depth-disc}$) are required to preserve them [S1].
*   **Noise in $x_{term}$:** Imperfect geometry reconstruction in the pre-trained NeRF leads to noise in $x_{term}$, requiring regularization via $\mathcal{L}_{offset}$ and $\mathcal{L}_{TV}$ [S1].
*   **Sharp Discontinuities:** Recoloring produces sharp discontinuities on the boundary of $E$ if adjacent cells are occupied. The growing queue mechanism and $\mathcal{L}_{smooth}$ mitigate this [S1].
*   **Palette Contribution:** Color palettes that do not contribute significantly should be removed after $1.5 \times 10^3$ iterations to maintain sparsity and efficiency [S1].
*   **Geometry-Aware Stylization Initialization:** For geometry-aware stylization, train without $\mathcal{L}_{style}, \mathcal{L}_{offset}, \mathcal{L}_{TV}$ for the first 1000 iterations to achieve a well-initialized palette $\hat{P}$ [S1].

## Subtopics

### Geometry-Aware Stylization with Depth Estimation
A stylization technique that respects the learned geometry of the pre-trained NeRF by performing depth estimation to counteract the removal of small structures common in naive 2D style transfer [S1]. It utilizes depth-guided total variation and depth discontinuity losses to preserve geometric details while applying style [S1].

## Operating characteristics
*   **Time Complexity:** The optimization of the LAENeRF module is lightweight as it operates on a sparse point cloud of ray terminations rather than the full volume. Training takes $\sim 20$ seconds for previews and $1 \times 10^5$ iterations total [S1]. Distillation takes no longer than 5 minutes on an NVIDIA RTX 4090 [S1].
*   **Memory Complexity:** Memory usage is dominated by the Instant-NGP hash grid and the edit grid ($128^3$), which is significantly lower than full-volume optimization methods [S1].
*   **Performance:** LAENeRF reduces background MSE error rates by 59% compared to PaletteNeRF with semantic guidance on the LLFF dataset [S1]. It introduces fewer background artifacts than Ref-NPR and models smooth transitions between original and edited content, a capability unique among compared methods [S1].
*   **Strengths:** Interactive local editing, photorealistic recoloring, non-photorealistic stylization, smooth transitions, low background artifacts, efficient computation [S1].
*   **Limitations:** Relies on the quality of the pre-trained NeRF's geometry (noise in $x_{term}$ if geometry is poor) [S1]. Requires user input for region selection [S1].
*   **When to use:** When local appearance editing (recoloring or stylization) is required with minimal background artifacts and smooth transitions.
*   **When not to use:** When global scene editing is needed, or when the pre-trained NeRF has poor geometry reconstruction.

## Background
LAENeRF was proposed by Lukas Radl, Michael Steiner, Andreas Kurz, and Markus Steinberger from Graz University of Technology [S1]. The method was published in arXiv:2312.09913v2 on March 25, 2024, and later in CVPR 2024 [S1]. It builds upon Instant-NGP for efficient rendering and NeRFShop for region selection [S1]. It is implemented on torch-ngp and evaluated on NeRF-Synthetic, LLFF, and mip-NeRF 360 datasets [S1].

## Sources
[S1] LAENeRF: Local Appearance Editing for Neural Radiance Fields, https://doi.org/10.1109/cvpr52733.2024.00475, fetched 2026-10-26
[S2] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
