# Rendered Depth-Guided Warping

Also called: Depth-Guided Warping

## Overview
Rendered Depth-Guided Warping is a consistency regularization technique used during NeRF training to enforce geometric coherence between observed and unobserved viewpoints without the computational cost of full-resolution rendering [S1, unofficial]. It generates a warped image from a source view using a low-resolution rendered depth map from a target view, enabling the computation of a consistency loss that encourages globally consistent geometry and appearance [S1, unofficial]. Unlike [Depth-Guided Sampling](7490-depth-guided-sampling.md) which uses depth to guide ray sampling, this method uses rendered depth to warp existing image data for loss calculation [S1, unofficial].

## How it works
**Inputs:**
*   Source view image $I_i$ at known viewpoint $i$ [S1, unofficial].
*   Target view (unseen) parameters: camera intrinsics $K$, rotation $R_{j \to i}$ relative to source [S1, unofficial].
*   NeRF model $f_\theta$ capable of outputting density $\sigma$ [S1, unofficial].

**Outputs:**
*   Warped image patch $I_{i \to j}$ aligned with the target view $j$ [S1, unofficial].

**Data Structures:**
*   **Strided Ray Grid:** A 2D grid of ray origins and directions sampled with stride $s$ (typically $s=2$) over the image dimensions [S1, unofficial].
*   **Low-Resolution Depth Map $D_j$:** Rendered depth values corresponding to the strided ray grid [S1, unofficial].
*   **Upsampled Depth Map:** Full-resolution depth map obtained via bilinear interpolation of $D_j$ [S1, unofficial].

**Steps:**
1.  **Strided Ray Sampling:** Sample rays on a grid with stride $s=2$ to create low-resolution patches [S1, unofficial].
2.  **Low-Resolution Depth Rendering:** For each sampled ray, query the NeRF model for density $\sigma$ and integrate along the ray to render a depth value $D_j(p_j)$ for pixel $p_j$ in the target view [S1, unofficial].
3.  **Depth Upsampling:** Upsample the low-resolution depth map $D_j$ back to the original image resolution using bilinear interpolation [S1, unofficial].
4.  **Coordinate Transformation:** For each pixel $p_j$ in the target view, compute the corresponding coordinate $p_{j \to i}$ in the source view $I_i$ using the camera intrinsics $K$, the viewpoint difference $R_{j \to i}$, and the rendered depth $D_j(p_j)$ [S1, unofficial]:
    $$ p_{j \to i} \sim K R_{j \to i} D_j(p_j) K^{-1} p_j $$
    where $\sim$ indicates approximate equality and $p_{j \to i}$ is a continuous coordinate [S1, unofficial].
5.  **Inverse Warping:** Use a differentiable sampler to extract color values from the source image $I_i$ at the projected coordinates $p_{j \to i}$ to generate the warped image $I_{i \to j}$ [S1, unofficial]:
    $$ I_{i \to j}(p_j) = \text{sampler}(I_i; p_{j \to i}) $$
6.  **Consistency Loss:** Compare the rendered target view image $I_j$ with the warped source image $I_{i \to j}$ using pixel-wise reconstruction loss or feature-level consistency loss to regularize the NeRF training [S1, unofficial].

**Parameters:**
*   **Stride $s$:** Typically set to 2 to balance computational efficiency and geometric accuracy [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Occlusions:** The warping assumes visibility. If the target view sees geometry occluded in the source view, the warped image will contain incorrect background colors [S1, unofficial]. The source notes mention an "Occlusion mask" in Figure 3, implying masking strategies may be required to handle these regions, though the specific algorithm is not detailed in the text [S1, unofficial].
*   **Depth Precision:** Errors in the rendered depth $D_j$ propagate directly into the projected coordinates $p_{j \to i}$ [S1, unofficial]. Low-resolution rendering may lack fine geometric detail, leading to misalignment artifacts [S1, unofficial].

## Subtopics

### Neural Warping
A strategy used in hybrid optimization pipelines (e.g., HO-Gaussian) to simulate images from virtual viewpoints by perturbing existing poses and querying colors from a grid-based volume [S2]. It helps align colors in multi-camera systems with limited overlapping areas and reduces overfitting to specific viewpoints [S2].
*   **Process:** Perturb an existing pose $[R_e, T_e]$ by rotating around the current position $x_p$ with random angles in the range $[-10, 10]$ to generate a virtual pose $[R_v, T_v]$ [S2].
*   **Projection:** Project the image point $p_i$ to the warped point $p_v$ using the intrinsic matrix $K$ and the virtual pose [S2]:
    $$ p_{v}=K(R_{v}(p_{i})+T_{v}) $$
*   **Color Query:** Obtain the color of the warped image at position $x_p$ and direction $d$ using the grid-based volume MLP $f_\theta$ [S2]:
    $$ c_v^\theta = f_\theta (x_p, d) $$

## Operating characteristics
*   **Computational Cost:** Significantly reduced compared to full-resolution rendering. Rendering a full image can take tens of seconds per iteration; strided sampling with $s=2$ reduces this cost to a fraction, enabling efficient consistency regularization during training [S1, unofficial].
*   **Accuracy:** The method relies on the accuracy of the rendered depth. Low-resolution depth may lack fine details, potentially causing warping artifacts in high-frequency regions [S1, unofficial].
*   **Use Case:** Best suited for few-shot NeRF training where geometric consistency is critical and full-resolution rendering is too slow for frequent consistency checks [S1, unofficial].

## Background
Rendered Depth-Guided Warping was introduced in the context of GeCoNeRF (Geometric Consistency NeRF) for few-shot neural radiance fields [S1, unofficial]. The method addresses the computational bottleneck of rendering full images for consistency regularization by leveraging low-resolution depth maps and inverse warping [S1, unofficial].

## Sources
[S1] GeCoNeRF: Few-Shot Neural Radiance Fields via Geometric Consistency, https://core.ac.uk/works/309257742, fetched 2026-11-01 (unofficial)
[S2] HO-Gaussian: Hybrid Optimization of 3D Gaussian Splatting for Urban Scenes, https://doi.org/10.1007/978-3-031-73027-6_2, fetched 2026-10-26
