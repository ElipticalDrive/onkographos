# DeSplat

Also called: Decomposed Gaussian Splatting

## Overview
DeSplat is a method for distractor-free novel view synthesis that decomposes the 3D Gaussian Splatting (3DGS) scene representation into two distinct sets of Gaussian primitives: static scene Gaussians ($G_s$) and per-view distractor Gaussians ($G_d$) [S1]. It solves the problem of transient occluders (e.g., pedestrians, vehicles) corrupting the static 3D reconstruction by separating them from the permanent scene geometry without relying on external semantic segmentation models or pre-trained neural networks [S1]. Unlike sibling methods that use foundation models for detection or complex neural appearance encoders for separation, DeSplat relies solely on volume rendering and photometric loss, allowing independent rendering of static elements and transient components while maintaining the speed of standard Gaussian splatting [S1].

## How it works
### Inputs and Outputs
**Inputs:**
- A set of $N$ calibrated images with known camera poses [G].
- A sparse point cloud from Structure-from-Motion (e.g., COLMAP) [S1].
- Hyperparameters: $\lambda_d$ (distractor opacity weight), $\lambda_{bg}$ (background loss weight), $T_\epsilon$ (background error threshold), warm-up steps (500), image downscaling factor (2) [S1].

**Outputs:**
- Static Gaussian set $G_s$ (optimized globally) [S1].
- Per-view Distractor Gaussian sets $G_d^{(j)}$ (optimized per view $j$) [S1].
- Rendered static image $\mathbf{c}_s$, distractor image $\mathbf{c}_d$, and composite image $\mathbf{c}_{comp}$ [S1].

### Data Structures
The method maintains two primary data structures for Gaussian primitives, following the standard 3DGS layout [S1, 9]:
1.  **Static Gaussians ($G_s$):**
    -   Global list of $N_s$ Gaussians.
    -   Fields per Gaussian: Position $\mathbf{p} \in \mathbb{R}^3$, Rotation $\mathbf{q} \in \mathbb{R}^4$ (quaternion), Scale $\mathbf{s} \in \mathbb{R}^3$, Opacity $\alpha \in \mathbb{R}$, Spherical Harmonics coefficients or RGB base color $\bar{c}_i$ [S1, 9].
2.  **Distractor Gaussians ($G_d$):**
    -   Stored as a list of lists or dictionary keyed by view index $j$ [G].
    -   $G_d^{(j)}$ contains $N_d^{(j)}$ Gaussians specific to camera view $j$ [G].
    -   Fields per Gaussian: Position $\mathbf{p} \in \mathbb{R}^3$, Rotation $\mathbf{q} \in \mathbb{R}^4$, Scale $\mathbf{s} \in \mathbb{R}^3$, Opacity $\alpha \in \mathbb{R}$, Color $\mathbf{c}_{RGB} \in \mathbb{R}^3$ (clamped to $[0, 1]$) [S1].
3.  **Embeddings (Optional, for appearance/background):**
    -   Per-image embedding $\mathbf{e}_j \in \mathbb{R}^{d_e}$ for view $j$ [S1].
    -   Per-Gaussian embedding $\mathbf{z}_i \in \mathbb{R}^{d_z}$ for static Gaussian $i$ [S1].
    -   MLPs: $f_\theta$ (appearance modulation), $f_{BG}$ (background SH prediction) [S1].

### Numbered Steps

1.  **Initialization:**
    -   Initialize $G_s$ using the sparse point cloud from COLMAP [S1].
    -   Initialize $G_d^{(j)}$ for each view $j$ by placing Gaussians on a 2D plane in front of the camera (near the camera plane) to model per-view transient effects [S1].
    -   Densify both sets via Adaptive Density Control (ADC) using cloning and splitting procedures identical to standard 3DGS [S1].

2.  **Optimization Loop (per step):**
    -   **Forward Pass (Rendering):**
        -   For each view $j$, rasterize $G_s$ and $G_d^{(j)}$ separately using standard alpha compositing [S1].
        -   Compute static color $\mathbf{c}_s$ and opacity $\alpha_s$ via standard volume rendering accumulation [S1].
        -   Compute distractor color $\mathbf{c}_d$ and opacity $\alpha_d$ similarly using $G_d^{(j)}$ [S1].
        -   **Composite Image:** Combine the two streams:
            $$ \mathbf{c}_{comp} = \mathbf{c}_d + (1 - \alpha_d) \mathbf{c}_s $$
            $$ \alpha_d = \sum_{i=1}^{N_d} \alpha_{d,i} \prod_{j=1}^{i-1} (1 - \alpha_{d,j}) $$
            where $\mathbf{c}_d$ and $\mathbf{c}_s$ are the pre-multiplied colors from the respective splats [S1, Eq. 4].
        -   **Appearance Modulation (if enabled):** If using appearance embeddings, modulate the base color $\mathbf{c}_i$ of static Gaussians before rasterization:
            $$ (\beta, \gamma) = f_\theta(\mathbf{e}_j, \mathbf{z}_i, \bar{c}_i) $$
            $$ \tilde{\mathbf{c}}_i = \gamma \odot \mathbf{c}_i + \beta $$
            where $\odot$ is element-wise multiplication, $\beta \in \mathbb{R}^3$ is shift, $\gamma \in \mathbb{R}^3$ is scale [S1, Eq. 6-7].
        -   **Background Rendering (if enabled):** Predict background SH coefficients $\mathbf{b}$ using $f_{BG}(\mathbf{e}_j)$ and render background color $\mathbf{c}_{BG}$ [S1, Eq. 8].

    -   **Loss Computation:**
        -   Compute photometric loss $L_{GS}$ (e.g., L1 + D-SSIM) between $\mathbf{c}_{comp}$ and Ground Truth (GT) image [S1].
        -   Compute Distractor Opacity Loss: $\lambda_d \alpha_d$ [S1, Eq. 12].
        -   Compute Background Loss $L_{bg}$ (if enabled):
            -   Calculate error $\epsilon(r) = ||\mathbf{c}_{comp}(r) - \mathbf{c}_{GT}(r)||$ for pixel $r$ [S1].
            -   Create binary mask $\tilde{e}(r) = 1$ if $\epsilon(r) \le T_\epsilon$, else 0 [S1].
            -   Smooth mask $M(r) = \tilde{e}(r) * B_{3\times3}$, where $B_{3\times3}$ is a $3\times3$ box filter [S1, Eq. 10-11].
            -   Define set $P = \{r | M(r) > 0.6\}$ [S1].
            -   Compute loss:
                $$ L_{bg} = \sum_{r \in P} |\alpha_d(r) + (1 - \alpha_d(r)) \cdot \alpha_s(r)| $$
                This encourages both static and distractor opacities to be low in background regions [S1, Eq. 9].
        -   Total Loss:
            $$ L = L_{GS} + \lambda_d \alpha_d + \lambda_{bg} L_{bg} $$
            [S1, Eq. 12].

    -   **Backward Pass & Update:**
        -   Compute gradients of $L$ with respect to all parameters ($G_s$, $G_d^{(j)}$, embeddings, MLPs) [G].
        -   Update parameters using an optimizer (e.g., Adam) [G].

    -   **Adaptive Density Control (ADC):**
        -   Apply cloning and splitting for both $G_s$ and $G_d^{(j)}$ based on gradient magnitudes, identical to standard 3DGS [S1].
        -   **Crucial Modification:** Apply opacity reset **only** to $G_s$. Disable opacity reset for $G_d^{(j)}$ to prevent confusion between occluders (distractors) and static objects [S1].

3.  **Rendering (Inference):**
    -   To render the static scene, use only $G_s$ and the composite formula with $\alpha_d=0, \mathbf{c}_d=0$, yielding $\mathbf{c}_s$ [S1].
    -   To render distractors, use only $G_d^{(j)}$, yielding $\mathbf{c}_d$ [S1].

### Formulas and Definitions
-   **Composite Color:** $\mathbf{c}_{comp} = \mathbf{c}_d + (1 - \alpha_d) \mathbf{c}_s$ [S1, Eq. 4].
-   **Distractor Opacity:** $\alpha_d = \sum_{i=1}^{N_d} \alpha_{d,i} \prod_{j=1}^{i-1} (1 - \alpha_{d,j})$ [S1, Eq. 4].
-   **Color Modulation:** $\tilde{\mathbf{c}}_i = \gamma \odot \mathbf{c}_i + \beta$ [S1, Eq. 7].
-   **Background Loss:** $L_{bg} = \sum_{r \in P} |\alpha_d(r) + (1 - \alpha_d(r)) \cdot \alpha_s(r)|$ [S1, Eq. 9].
-   **Mask:** $M(r) = (\tilde{e}(r) * B_{3\times3})$ where $\tilde{e}(r) = \mathbb{I}(\epsilon(r) \le T_\epsilon)$ [S1, Eq. 10-11].
-   **Total Loss:** $L = L_{GS} + \lambda_d \alpha_d + \lambda_{bg} L_{bg}$ [S1, Eq. 12].

### Parameters
-   $\lambda_d$: Weight for distractor opacity regularization. Not given in the sources.
-   $\lambda_{bg}$: Weight for background loss. Not given in the sources.
-   $T_\epsilon$: Threshold for background error. Not given in the sources.
-   Warm-up steps: 500 [S1].
-   Image downscaling: Factor of 2 at beginning of training [S1].

### Edge Cases and Pitfalls
-   **Opacity Reset:** Applying opacity reset to distractor Gaussians causes confusion between occluders and static objects, leading to poor separation [S1]. It must be disabled for $G_d$.
-   **Color Representation:** Distractor Gaussians must use RGB clamping $[0, 1]$ instead of Spherical Harmonics (SH) with sigmoid. SH is unsuitable for per-view distractors as it requires view-dependent compensation, and sigmoid struggles with extreme colors (white/black) [S1].
-   **Initialization:** Distractors must be initialized near the camera plane to model view-specific transients effectively [S1].

## Operating characteristics
-   **Time Complexity:** Similar to standard 3DGS per step, requiring two separate rasterization passes ($G_s$ and $G_d^{(j)}$) and alpha compositing [S1].
-   **Memory Complexity:** Increases memory usage by storing $G_d^{(j)}$ per view; size depends on distractor density, which remains sparse [S1].
-   **Measured Performance:** Evaluated on RobustNeRF (Statue, Android, Crab, Yoda scenes), DrivingGaussian, and Photo Tourism datasets; achieves comparable results to prior distractor-free approaches without sacrificing rendering speed [S1].
-   **Scaling:** Not given in the sources.
-   **Strengths:** No external semantic models required; fully compatible with appearance modeling and background models; explicit separation of static and dynamic elements [S1].
-   **Limitations:** Requires per-view optimization for distractors, increasing training time slightly compared to single-set 3DGS [S1].
-   **When to use:** Distractor-free novel view synthesis in dynamic scenes where transient occluders must be separated from static geometry [S1].
-   **When not:** When external semantic priors are available and distractors require complex semantic separation rather than view-specific transient modeling [S1].

## Background
DeSplat was proposed by Aalto University researchers (authors include Antti Aalto, et al.) and presented at CVPR 2025 [S1]. It addresses the limitations of existing distractor-free methods that rely on pre-trained semantic models (e.g., ResNet-18, foundation models) which introduce computational overhead [S1]. The method builds on the 3DGS framework [9] and draws inspiration from 2D image Gaussian Splatting [48] for initializing view-specific distractors. It is designed for novel view synthesis in dynamic scenes, such as autonomous driving scenarios (e.g., DrivingGaussian dataset) and photo-tourism datasets [S1].

## Sources
[S1] DeSplat: Decomposed Gaussian Splatting for Distractor-Free Rendering, https://doi.org/10.1109/CVPR52734.2025.00076, fetched 2026-11-03
