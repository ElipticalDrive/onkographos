# PI-NeRF

Also called: Partial-Invertible Neural Radiance Fields

## Overview
PI-NeRF is a framework that integrates NeRF with Invertible Neural Networks (INN) to directly output the 6-DoF camera pose from a single image without pose initialization or iterative optimization [S1]. It solves the inverse problem of view synthesis by modifying the radiance field to output high-dimensional features, which are then inverted to recover ray representations [S1]. Unlike sibling methods such as [NeRF2Points](12125-nerf2points.md), [GazeNeRF](12127-gazenerf.md), and [HeadNeRF](12332-headnerf.md) which often rely on iterative optimization or specific geometric constraints, PI-NeRF achieves direct pose estimation by inverting the rendering process [S1].

## How it works
PI-NeRF consists of three main components: a feature-based NeRF (NeRF-Feat), an Invertible Neural Network Module (INM), and an image encoder (ResUNet) [S1].

### Inputs and Outputs
*   **Inputs:** A single RGB image $I$ with unknown pose $(R, t)$ relative to the scene.
*   **Outputs:** The 6-DoF camera pose $(R, t)$ [S1].

### Data Structures
1.  **NeRF-Feat:** An MLP-based radiance field.
    *   Input: 3D point $\mathbf{x} \in \mathbb{R}^3$ and viewing direction $\mathbf{d} \in \mathbb{R}^3$.
    *   Output: High-dimensional feature vector $\mathbf{f} \in \mathbb{R}^k$ (where $k$ is the feature dimension) and density $\sigma \in \mathbb{R}$ [S1].
    *   Note: This replaces the standard RGB output $\mathbf{c} \in \mathbb{R}^3$ with $\mathbf{f}$ [S1].
2.  **INM (Invertible Neural Network Module):** Composed of two INNs operating in forward and reverse directions [S1].
    *   Forward Path Input: Ray feature $\mathbf{f}_{ray}$ and ray representation $\mathbf{r}_{rep}$ (encoding ray geometry/pose information).
    *   Forward Path Output: Pixel feature $\mathbf{f}_{pix}$ [S1].
    *   Reverse Path Input: Pixel feature $\mathbf{f}_{pix}$.
    *   Reverse Path Output: Recovered ray feature $\mathbf{f}_{ray}$ and ray representation $\mathbf{r}_{rep}$ [S1].
3.  **ResUNet:** A U-Net based encoder [S1].
    *   Input: RGB Image $I$.
    *   Output: Pixel feature map $\mathbf{F}_{img}$ [S1].

### Numbered Steps

**Training Phase:**
1.  **NeRF-Feat Training:** Train the standard NeRF architecture to output high-dimensional features $\mathbf{f}$ instead of RGB colors, using volume rendering to aggregate features along rays [S1].
2.  **Image Encoding:** For a given image $I$, use ResUNet to extract a pixel feature map $\mathbf{F}_{img}$ [S1].
3.  **Feature Sampling:** Sample a set of pixel features $\mathbf{f}_{pix}$ from $\mathbf{F}_{img}$ corresponding to image pixels [S1].
4.  **Reverse INM:** Feed $\mathbf{f}_{pix}$ into the reverse path of the INM to recover the corresponding ray features $\mathbf{f}_{ray}$ and ray representations $\mathbf{r}_{rep}$ [S1].
5.  **Forward INM:** Feed the recovered $\mathbf{f}_{ray}$ and $\mathbf{r}_{rep}$ into the forward path of the INM to generate predicted pixel features $\hat{\mathbf{f}}_{pix}$ [S1].
6.  **Loss Computation:** Compute the Barlow Twins (BT) loss between the ResUNet-extracted features $\mathbf{f}_{pix}$ and the INM-recovered features $\hat{\mathbf{f}}_{pix}$ [S1].
    *   The Barlow Twins loss encourages the extracted features to be invariant to the inversion process, ensuring the ResUNet learns features suitable for the reverse INM [S1].
7.  **Optimization:** Update ResUNet and INM parameters to minimize the BT loss [S1].

**Inference Phase:**
1.  **Image Encoding:** Use the trained ResUNet to extract the pixel feature map $\mathbf{F}_{img}$ from the input image $I$ [S1].
2.  **Feature Sampling:** Sample a set of pixel features $\mathbf{f}_{pix}$ from $\mathbf{F}_{img}$ [S1].
3.  **Reverse INM:** Feed $\mathbf{f}_{pix}$ into the reverse path of the trained INM to recover the ray representations $\mathbf{r}_{rep}$ [S1].
4.  **Pose Estimation:** Use the recovered ray representations $\mathbf{r}_{rep}$ (which encode the ray directions and origins relative to the camera) and the known 3D points from the NeRF scene to calculate the camera pose $(R, t)$ using a Perspective-n-Point (PnP) algorithm [S1].

### Formulas
*   **Volume Rendering of Features:**
    $$
    \hat{\mathbf{F}}(r) = \sum_{i=1}^{N} T_i \alpha_i \mathbf{f}(x_i)
    $$
    where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ is the accumulated transmittance, $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ is the opacity weight, and $\mathbf{f}(x_i)$ is the high-dimensional feature output by NeRF-Feat at sample point $x_i$ [S1, Parent].

*   **Barlow Twins Loss:**
    The loss is computed between the feature vector from ResUNet ($\mathbf{f}_{pix}$) and the feature vector from the forward INM ($\hat{\mathbf{f}}_{pix}$) [S1]. The specific mathematical formulation of the Barlow Twins loss is not detailed in the source, but it is used to align the distributions of these features [S1].

### Parameters
*   **Feature Dimension ($k$):** Not given in the sources.
*   **NeRF MLP Depth/Width:** Not given in the sources.
*   **INM Architecture:** Not given in the sources.
*   **ResUNet:** Standard U-Net architecture [S1].

### Edge Cases and Pitfalls
*   **PnP Degeneracy:** The PnP algorithm requires a sufficient number of non-collinear 3D-2D correspondences. If the recovered ray representations are inaccurate or sparse, PnP may fail [S1].
*   **Feature Ambiguity:** The high-dimensional features must be unique enough to distinguish between different rays/poses. If features are ambiguous, the INM inversion may yield incorrect ray representations [S1].
*   **No Initial Pose:** The method does not require an initial pose, but this also means it cannot refine a nearby pose. It relies entirely on the accuracy of the direct inversion [S1].

## Operating characteristics
*   **Speed:** PI-NeRF is two orders of magnitude faster than existing NeRF-based pose estimation methods (e.g., pNeRF takes >5 seconds, iNeRF is slower) [S1].
*   **Accuracy:** Competitive accuracy compared to iterative methods, and outperforms NeRF-free absolute pose regression methods by a large margin [S1].
*   **Initialization:** Does not require an initial pose [S1].
*   **Strengths:** Direct pose estimation, no iterative optimization, real-time capable, robust to initial pose errors.
*   **Limitations:** Requires training of NeRF-Feat, INM, and ResUNet. Accuracy depends on the quality of the high-dimensional features and the invertibility of the INM [S1].
*   **When to use:** When fast, initialization-free camera pose estimation is needed from a single image of a NeRF-represented scene.
*   **When not to use:** When iterative refinement is required for higher precision, or when the scene is not well-represented by a NeRF.

## Background
PI-NeRF was proposed by Zhihao Li, Kexue Fu, Haoran Wang, and Manning Wang in 2023 [S1]. It addresses the limitations of existing NeRF-based pose estimation methods like pNeRF and iNeRF, which rely on synthesizing images from multiple poses or iterative optimization, making them time-consuming and sensitive to initial poses [S1]. PI-NeRF is the first work to use NeRF for direct pose estimation by inverting the rendering process [S1]. It builds on the principles of [Semantic and Feature Branches](12095-semantic-and-feature-branches.md) by using high-dimensional feature fields for inverse problems [S1].

## Sources
[S1] PI-NeRF: A Partial-Invertible Neural Radiance Fields for Pose Estimation, https://doi.org/10.1145/3581783.3612590, fetched 2026-11-08
