# ViP-NeRF

Also called: Visibility Prior NeRF

## Overview
ViP-NeRF is a training procedure for Neural Radiance Fields (NeRF) that introduces visibility priors to supervise density estimation, specifically designed to improve rendering quality under sparse input view conditions (fewer than 50–100 views) [S1]. It prevents artifacts caused by insufficient training data by estimating whether pixels in a main view are visible in secondary views, using this information to regularize the NeRF density field [S2]. Unlike methods that rely on pre-trained geometric priors or diffusion models, ViP-NeRF computes these visibility priors using planar scanned volumes without requiring any pre-training [S2].

## How it works
**Inputs:**
*   A set of input images with known camera poses [G].
*   For each "main view" image, a set of "secondary view" images [G].
*   A NeRF model parameterized by $\theta$, mapping 3D coordinates $\mathbf{x}$ and viewing directions $\mathbf{d}$ to density $\sigma$ and color $\mathbf{c}$ [G].

**Outputs:**
*   A trained NeRF model [G].
*   A visibility prior map for the main view [S2].

**Data Structures:**
*   **Plane Scan Volume:** A list of 2D error maps. For a given main view and secondary view pair, this volume contains error maps corresponding to different depth planes [S2].
*   **Visibility Prior Map:** A 2D map of the same resolution as the main view image, where each pixel value indicates the likelihood of visibility (derived from thresholded minimum error) [S2].

**Algorithm:**

1.  **Visibility Regularization Setup:** The method integrates a visibility regularization term into the NeRF training loss. This term uses the visibility prior map to supervise the density $\sigma$ of the NeRF [S2].

2.  **Visibility Prior Estimation:** For a specific main view $I_{\text{main}}$ and a secondary view $I_{\text{sec}}$, estimate the visibility of each pixel in $I_{\text{main}}$ by checking if it is visible in $I_{\text{sec}}$ [S2].

3.  **Effective Prediction of Visibility (Planar Scan):**
    a.  Define a set of depth planes $D = \{d_1, d_2, \dots, d_k\}$ relative to the camera frame of the secondary view [G].
    b.  For each depth plane $d_i \in D$:
        i.  Warp the image $I_{\text{sec}}$ to the coordinate system of the main view using the known camera poses and the depth $d_i$. Let the warped image be $I_{\text{warped}}^{(i)}$ [G].
        ii. Compute an error map $E^{(i)}$ between the main view $I_{\text{main}}$ and the warped image $I_{\text{warped}}^{(i)}$. The error is measured using pixel-wise differences (e.g., L1 or L2 norm) [G].
    c.  Find the minimum error for each pixel across all depth planes:
        $$ E_{\min}(u, v) = \min_{i} E^{(i)}(u, v) $$
        where $(u, v)$ are pixel coordinates in the main view [S2].
    d.  Threshold the minimum error map $E_{\min}$ to obtain the visibility prior map $V_{\text{map}}$:
        $$ V_{\text{map}}(u, v) = \begin{cases} 1 & \text{if } E_{\min}(u, v) < \tau \\ 0 & \text{otherwise} \end{cases} $$
        where $\tau$ is a predefined threshold [S2].

4.  **NeRF Training with Visibility Prior:**
    a.  Render the main view using the current NeRF model [G].
    b.  Compute the standard photometric loss between the rendered image and $I_{\text{main}}$ [G].
    c.  Compute the visibility regularization loss. For pixels where $V_{\text{map}}(u, v) = 1$, enforce that the accumulated density along the ray should correspond to a visible surface (e.g., by penalizing high density in empty space or ensuring density peaks align with the visible geometry). For pixels where $V_{\text{map}}(u, v) = 0$, the density can be suppressed if the geometry is not consistent with the secondary view's occlusion [S2].
    d.  Update NeRF parameters $\theta$ using gradient descent [G].

**Parameters:**
*   $\tau$: Threshold for visibility prior map. Value not given in the sources.
*   $D$: Set of depth planes. The number of planes and their range are not given in the sources [S2].

**Edge Cases and Pitfalls:**
*   Not given in the sources.

## Subtopics

### Visibility-Based Interpolation
Interpolation methods that weigh Block-NeRFs based on per-pixel or per-image predicted visibility to improve sharpness of distant areas [S3]. On the Mission Bay dataset, Pixelwise Visibility achieved a PSNR of 27.39 and Imagewise Visibility achieved a PSNR of 27.41 [S3]. These methods produce sharper reconstructions of far-away areas but are prone to temporal inconsistency, making them suitable only for still images [S3].

## Operating characteristics
*   **Time and Memory Complexity:** The method requires computing planar scanned volumes for each main-secondary view pair during training, which adds computational overhead. The complexity depends on the number of depth planes and the resolution of the images [S2].
*   **Measured Performance:** On the RealEstate-10K dataset, ViP-NeRF achieves a PSNR index 3–4 points higher than the latest paper in 2, 3, and 4 input views [S2].
*   **Strengths:** Improves rendering quality under sparse view conditions without pre-training [S2].
*   **Limitations:** Requires careful tuning of the threshold $\tau$ and the set of depth planes. Performance may degrade if camera poses are inaccurate [S2].
*   **When to use:** When training NeRFs with very few input views (e.g., 2–4 views) and no external geometric priors are available [S2].
*   **When not to use:** When temporal consistency is required for video rendering, as related visibility-based interpolation methods are prone to temporal inconsistency [S3].

## Background
ViP-NeRF was introduced to address the challenge of novel view synthesis with sparse input views. It builds on the idea that visibility information from multiple views can provide a reliable estimate to supervise the density of the NeRF [S2]. The method was proposed in the context of improving the rendering quality of synthesized images from new viewpoints under sparse input view conditions [S2].

## Sources
[S1] Neural Radiance Field-based Visual Rendering: a Comprehensive Review, https://ieeexplore.ieee.org/abstract/document/11455380, fetched 2026-10-30
[S2] Methods and Strategies for Improving the Novel View Synthesis Quality of Neural Radiation Field, https://doi.org/10.1109/access.2024.3382997, fetched 2026-10-30
[S3] Block-NeRF: Scalable Large Scene Neural View Synthesis, https://doi.org/10.1109/cvpr52688.2022.00807, fetched 2026-10-26
