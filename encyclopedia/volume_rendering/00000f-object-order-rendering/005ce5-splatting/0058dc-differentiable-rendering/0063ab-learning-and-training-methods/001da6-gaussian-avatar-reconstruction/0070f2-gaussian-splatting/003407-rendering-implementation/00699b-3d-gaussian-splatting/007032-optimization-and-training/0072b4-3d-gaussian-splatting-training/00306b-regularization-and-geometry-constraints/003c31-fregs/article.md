# FreGS

Also called: Frequency-aware regularization for 3DGS, Progressive frequency regularization

## Overview
FreGS addresses the over-reconstruction and high-frequency artifacts common in 3D Gaussian Splatting (3DGS) by applying progressive frequency regularization to the Gaussian primitives [S1, unofficial][S2]. Unlike sibling methods that focus on pixel-level gradients or geometric depth consistency, FreGS operates in the frequency domain to suppress noise and prevent the model from fitting high-frequency noise present in training data [S1, unofficial][S2]. It builds on [Regularization and Geometry Constraints](6269-regularization-and-geometry-constraints.md) by defining a regularization term based on spectral discrepancies.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Current set of 3D Gaussians (position, covariance, opacity, color), camera parameters, training images, and their ground truth counterparts.
*   **Outputs:** Updated Gaussian parameters with reduced high-frequency artifacts.

**Data Structures:**
*   Standard 3DGS Gaussian attributes: mean $\mu$, covariance $\Sigma$, opacity $\alpha$, spherical harmonics coefficients $c$.
*   Rendered image $I_{render}$ and Ground Truth image $I_{gt}$.

**Method Steps:**
1.  **Render:** Compute the rendered image $I_{render}$ from the current Gaussian set using standard differentiable rasterization.
2.  **Frequency Transform:** Apply a Fourier Transform (or similar frequency decomposition) to both $I_{render}$ and $I_{gt}$ to obtain their frequency spectra $F_{render}$ and $F_{gt}$ [S2].
3.  **Compute Frequency Discrepancy:** Calculate the difference between the frequency spectra. The regularization loss $\mathcal{L}_{freq}$ is derived from the high-frequency components where discrepancies are most indicative of overfitting [S2].
4.  **Progressive Application:** Apply the regularization progressively during training. This likely involves adjusting the weight or the frequency band considered as training progresses to stabilize convergence [S1, unofficial].
5.  **Loss Combination:** Combine with the standard photometric loss (L1 + SSIM) as per the parent framework:
    $$ \mathcal{L}_{total} = \mathcal{L}_{photo} + \lambda \mathcal{L}_{freq} $$
    where $\lambda$ is the regularization weight [Parent].
6.  **Backpropagation:** Compute gradients with respect to Gaussian parameters and update them. Note that gradient updates for scaling and rotation may require stabilization techniques (e.g., rescaling gradients) as noted in related works like Faster-GS, though FreGS specifically targets frequency domain errors [S1, unofficial].

**Parameters:**
*   $\lambda$: Regularization weight. Not given in the sources.
*   Frequency band cutoff: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Over-smoothing:** Excessive regularization may remove fine details. The "progressive" nature aims to mitigate this [S1, unofficial].
*   **Precision:** Frequency domain operations require careful handling of numerical precision to avoid artifacts in the gradient computation [G].

## Subtopics

### Frequency Regularization
The broader category of methods using spectral analysis. FreGS is a specific instance using progressive application. [Link](6130-frequency-regularization.md)

### Smoothed 2D Supervision
Uses spatial smoothing rather than explicit frequency domain analysis. [Link](7213-smoothed-2d-supervision.md)

### 3D Half-Gaussian Splatting Covariance Transformation and Regularization
Focuses on covariance matrix regularization in 3D space. [Link](9150-3d-half-gaussian-splatting-covariance-transformation-and-reg.md)

### Hard and Soft Depth Regularization
Uses depth maps for geometric constraints. [Link](9178-hard-and-soft-depth-regularization.md)

### Cross-scale geometric adaptation
Adapts to different scales of geometry. [Link](10796-cross-scale-geometric-adaptation.md)

### Geometry Regularization
General geometric constraints. [Link](14222-geometry-regularization.md)

## Operating characteristics
*   **Time Complexity:** Adds overhead for Fourier transforms and frequency domain calculations per iteration. Not given in the sources.
*   **Memory Complexity:** Requires storing frequency spectra. Not given in the sources.
*   **Strengths:** Effectively reduces high-frequency artifacts and over-reconstruction [S1, unofficial][S2].
*   **Limitations:** May require careful tuning of the progressive schedule and regularization weight. Not given in the sources.
*   **When to use:** When training results in noisy, over-fitted reconstructions with high-frequency artifacts.
*   **When not to use:** When fine high-frequency details are critical and the model is under-fitting.

## Background
FreGS was introduced to address the issue of over-reconstruction in 3D Gaussian Splatting [S2]. It applies progressive frequency regularization to primitives to prevent overfitting and high-frequency artifacts during training [S1, unofficial]. The method focuses on the frequency domain to regularize discrepancies between rendered and ground truth images [S2].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] 3DGS.zip: A survey on 3D Gaussian Splatting Compression Methods, https://arxiv.org/abs/2407.09510v5, fetched 2026-10-26
