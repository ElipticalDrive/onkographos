# Reprojection Confidence Measurement
Also called: Warping-based confidence
## Overview
Reprojection Confidence Measurement evaluates depth accuracy and locates outliers by warping pixels from a source view to a target view and comparing the reconstructed appearance against the actual target image [S1, unofficial]. It distinguishes itself from sibling methods like [Probabilistic Depth Testing](11488-probabilistic-depth-testing.md) by explicitly constructing confidence maps at multiple semantic levels (pixel, patch, and feature) to identify outliers caused by moving objects, ill-posed regions, and occlusions [S1, unofficial].
## How it works
**Inputs and Outputs:**
*   **Inputs:** Source image $I_s$, source depth map $D_s$, target image $I_t$, source camera parameters $P_s$, target camera parameters $P_t$ [S1, unofficial].
*   **Outputs:** Confidence maps $C_{rgb}$, $C_{ssim}$, $C_{vgg}$ [S1, unofficial].

**Data Structures:**
*   Pixel coordinates $X = (x, y, d)$, where $(x,y)$ are 2D image coordinates and $d$ is the depth value [S1, unofficial].
*   Camera parameters $P$ containing intrinsic and extrinsic matrices [S1, unofficial].

**Steps:**
1.  **Unproject:** Convert source pixel $X_s = (x_s, y_s, d_s)$ to 3D coordinates using the inverse warping function $\psi^{-1}$ and source camera parameters $P_s$ [S1, unofficial].
2.  **Reproject:** Project the 3D point to the target image plane using the warping function $\psi$ and target camera parameters $P_t$ to obtain target pixel coordinates $X_t = (x_t, y_t, d_t)$ [S1, unofficial]:
    $$X_t = \psi(\psi^{-1}(X_s, P_s), P_t)$$
3.  **Warp Image:** Sample the target image $I_t$ at the reprojected coordinates $X_t$ to generate the warped source image $\hat{I}_s$:
    $$\hat{I}_s = I_t(X_t)$$
4.  **Compute RGB Confidence:** Calculate the pixel-wise absolute difference between the original source image $I_s$ and the warped image $\hat{I}_s$:
    $$C_{rgb} = 1 - |I_s - \hat{I}_s|$$
5.  **Compute SSIM Confidence:** Calculate the Structural Similarity Index Measure between $I_s$ and $\hat{I}_s$ to measure patch structure similarity:
    $$C_{ssim} = \text{SSIM}(I_s, \hat{I}_s)$$
6.  **Compute VGG Confidence:** Extract features $F_s$ from $I_s$ and $\hat{F}_s$ from $\hat{I}_s$ using a pre-trained VGG network, then compute the feature-level similarity:
    $$C_{vgg} = 1 - ||F_s - \hat{F}_s||$$
    where $\hat{F}_s = F_t(X_t)$ refers to the feature reprojection [S1, unofficial].

**Parameters:**
*   **VGG Network:** Pre-trained VGG feature extractor (Simonyan & Zisserman, 2015) [S1, unofficial].
*   **SSIM Window:** Standard SSIM calculation (Wang et al., 2004) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Occlusions:** Pixels occluded in the target view will result in incorrect warping, leading to low confidence scores [S1, unofficial].
*   **Moving Objects:** Dynamic objects violate the static scene assumption, causing reprojection errors [S1, unofficial].
*   **Ill-posed Regions:** Areas with insufficient texture or depth ambiguity may produce unreliable depth $d_s$, leading to erroneous reprojection [S1, unofficial].
*   **Precision:** The warping process relies heavily on the accuracy of $D_s$; depth outliers directly propagate to confidence errors [S1, unofficial].

## Subtopics

### [Geometry Confidence Measurement](15151-geometry-confidence-measurement.md)
Measures geometry consistency of depths and flows across views to identify outliers in multi-view stereo and neural rendering pipelines. It sets itself apart from sibling methods by explicitly combining geometric reprojection errors with optical flow consistency, rather than relying solely on photometric or feature similarity. It builds on Reprojection Confidence Measurement by adding a flow-based constraint to detect motion inconsistencies that reprojection alone might miss [G]. Choose it when scenes contain significant non-rigid motion or require explicit flow constraints [G].

## Operating characteristics
**Time and Memory Complexity:**
*   Time complexity: Not given in the sources.
*   Memory complexity: Not given in the sources.

**Measured Performance:**
*   Not given in the sources.

**Strengths and Limitations:**
*   **Strengths:** The receptive fields of the confidence maps expand from pixels to local patches and non-local regions, constructing robust measurements [S1, unofficial].
*   **Limitations:** Relies on accurate depth maps for unprojection; sensitive to dynamic scenes and occlusions [S1, unofficial].

**When to Use:**
*   Use when jointly optimizing neural radiance fields (e.g., S-NeRF) and needing to filter noisy depth supervisions [S1, unofficial].

**When Not to Use:**
*   Not suitable for scenes with significant non-rigid motion without additional flow constraints [S1, unofficial]. (See [Geometry Confidence Measurement](15151-geometry-confidence-measurement.md) for flow-based alternatives.)
## Background
**Origin:**
*   Introduced in S-NeRF: Neural Radiance Fields for Street Views [S1, unofficial].

**History and Development:**
*   The method was designed to address the quantity of outliers due to moving objects, ill-posed regions, and occlusions in street view data [S1, unofficial].

**Usage:**
*   Used in the training of S-NeRF to provide robust confidence measurements for depth supervision [S1, unofficial].
## Sources
[S1] S-NeRF: Neural Radiance Fields for Street Views, https://doi.org/10.48550/arxiv.2303.00749, fetched 2026-10-29 (unofficial)
