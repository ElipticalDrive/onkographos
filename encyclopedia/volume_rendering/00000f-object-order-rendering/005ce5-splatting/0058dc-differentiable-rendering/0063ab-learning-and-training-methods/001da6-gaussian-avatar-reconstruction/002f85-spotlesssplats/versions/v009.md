# SpotlessSplats
## Overview
SpotlessSplats is a robustness-enhancing method for Gaussian Avatar Reconstruction that improves reconstruction quality in casual captures by detecting and ignoring transient distractors [S1][S2, unofficial]. Unlike SWAG, which trains unsupervised transient Gaussians to ignore occluders, SpotlessSplats operates in a richer, pre-trained feature space to filter noise [S1][S2, unofficial].

## How it works
**Inputs and Outputs:** Not given in the sources.
**Data Structures:** Not given in the sources.
**Steps:**
1. **Feature Extraction:** Project deformed Gaussians into a pre-trained feature space [S1][S2, unofficial].
2. **Outlier Detection:** Identify Gaussians corresponding to transient distractors by detecting outliers in this feature space [S1][S2, unofficial].
3. **Filtering:** Ignore or suppress the contribution of detected outlier Gaussians during the differentiable rendering process [S1][S2, unofficial].
**Formulas:** Specific mathematical formulations for the feature extractor or outlier detection function are not provided in the sources [S1][S2, unofficial]. The method modifies the standard differentiable rendering pipeline to exclude identified outliers [S1][S2, unofficial].
**Parameters:** Feature space dimension, outlier threshold, and feature extractor architecture are not given in the sources.
**Edge Cases and Pitfalls:** The method assumes avatar geometry is stable and distinct from transient elements; severe occlusion or extreme pose changes may lead to incorrect outlier classification [G]. Effectiveness depends on the quality of the pre-trained feature space [S1][S2, unofficial].

## Subtopics

### Point Rasterization
Point Rasterization is a neural point rendering technique from DGNR, not explicitly part of SpotlessSplats, but related to the rendering pipeline [S3].
*   **Definition:** Density points from space $P$ are projected using camera model $C$ to create images of dimensions $W \times H$, assigning points passing the depth test to neural density features [S3].
*   **Steps:**
    1. Convert real-valued projection $Q_s(p_r)$ to integer pixel coordinates $q_r$ by rounding to nearest integer: $q_r = \lfloor Q_s(p_r) \rceil$ [S3].
    2. Encode pixels $q_r$ into neural density features $\tau$ through a $1 \times 1$ convolution kernel, encoding local 3D scene context around $p_r$ [S3].
*   **Formula:** World point $p_r$ is projected to layer $l$ image space:
    $$ Q_s(p_r) = \frac{1}{2^l} C(R(p_r) + T) $$
    where $C$ is the camera model, and $(R, T)$ is the rigid transformation from world to camera space [S3].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** SpotlessSplats yields improved reconstruction quality in casual captures by ignoring transient distractors [S1][S2, unofficial]. It builds on SWAG by further enhancing robustness [S1][S2, unofficial].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Enhanced robustness to transient distractors in casual captures; improved reconstruction quality [S1][S2, unofficial].
*   **Limitations:** Relies on a pre-trained feature space; specific implementation details (thresholds, feature dimensions) are not provided [S1][S2, unofficial].
*   **When to use:** When capturing casual videos with transient distractors (e.g., moving people, shadows) that degrade standard Gaussian Splatting reconstruction [S1][S2, unofficial].
*   **When not to use:** Not given in the sources.

## Background
SpotlessSplats was introduced as an extension to 3D Gaussian Splatting for robust reconstruction [S1][S2, unofficial]. It builds on SWAG, which uses unsupervised transient Gaussians, by detecting outliers in a richer, pre-trained feature space [S1][S2, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
[S3] DGNR: Density-Guided Neural Point Rendering of Large Driving Scenes, https://doi.org/10.1109/tase.2024.3410891, fetched 2026-10-27
