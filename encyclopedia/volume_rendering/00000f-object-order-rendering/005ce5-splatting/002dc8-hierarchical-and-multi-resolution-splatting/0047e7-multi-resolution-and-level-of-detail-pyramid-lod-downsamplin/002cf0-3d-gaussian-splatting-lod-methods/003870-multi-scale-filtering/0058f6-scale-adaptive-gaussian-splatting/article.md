# Scale-Adaptive Gaussian Splatting

Also called: SA-GS

## Overview
Scale-Adaptive Gaussian Splatting (SA-GS) is a training-free, test-time plugin for 3D Gaussian Splatting that improves anti-aliasing by applying 2D scale-adaptive filters to each Gaussian primitive [S2, unofficial]. It solves the mismatch between Gaussian scales learned during training and the sampling frequency encountered at test time, which causes aliasing artifacts when zooming out or rendering at low resolutions [S2, unofficial]. Unlike [Mip-Splatting](6063-mip-splatting.md), which modifies the training procedure and uses 3D smoothing/2D Mip filters unaware of testing frequency, SA-GS operates at test-time and uses filters informed by testing frequency [S2, unofficial].

## How it works
**Inputs:** Pre-trained 3D Gaussian field (positions, covariances, opacities, colors); camera parameters (view matrix, projection matrix, focal length); rendering resolution $(W, H)$ [G].
**Outputs:** Rendered image [G].

**Data Structures:**
*   **Gaussian Primitive:** Defined by mean position $\mu$, 3D covariance $\Sigma_{3D}$ (or scale/rotation), opacity $\alpha$, and spherical harmonic coefficients for color [G].
*   **Projected Covariance:** $\Sigma_{2D}$, a $2 \times 2$ matrix representing the Gaussian's spread in screen space [G].

**Steps:**

1.  **Calculate Testing Frequency:** Determine the effective sampling frequency based on the rendering resolution and camera distance [G].
2.  **Project and Regularize Covariance:**
    *   Project the 3D Gaussian to 2D screen space to obtain $\Sigma_{2D}$ [G].
    *   Apply a 2D scale-adaptive low-pass filter by adding a variance term $\sigma^2_{test}$ to the projected covariance:
        $$ \Sigma_{reg} = \Sigma_{2D} + \sigma^2_{test} I $$
        where $I$ is the $2 \times 2$ identity matrix and $\sigma^2_{test}$ is derived from the testing frequency (resolution and camera distance) [G].
3.  **Adjust Opacity:**
    *   To preserve the integral (total mass) of the Gaussian after blurring, scale the opacity $\alpha$ by a normalization factor $C$:
        $$ C = \frac{\sqrt{|\Sigma_{2D}|}}{\sqrt{|\Sigma_{reg}|}} $$
        $$ \alpha_{new} = \alpha_{old} \cdot C $$
        This ensures the Gaussian's contribution remains consistent despite the scale adaptation [G].
4.  **Integrate within Pixels:**
    *   Instead of simple point sampling, integrate the projected 2D Gaussian within each pixel area [S2, unofficial]. This acts as a limiting case of super-sampling, handling sampling rates smaller than the scene frequency [S2, unofficial].
    *   [G] The integration can be approximated by evaluating the cumulative distribution function or using a fixed number of samples within the pixel bounds weighted by the Gaussian PDF.

**Parameters:**
*   $\sigma^2_{test}$: Adaptive variance term. Not given in the sources as a fixed formula, but described as being informed by testing frequency (resolution and camera distance) [S1][S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Jaggedness without Filtering:** Super-sampling and integration are only effective when the scale-adaptive filtering is activated; without it, scale inconsistency leads to jaggedness at low sampling rates [S2, unofficial].
*   **Scale Mismatch:** If the Gaussian scale does not match the testing frequency, high-frequency artifacts appear [S2, unofficial].

## Subtopics

### [Alias-free 4D Gaussian Splatting](9872-alias-free-4d-gaussian-splatting.md)
Eliminates high-frequency artifacts in dynamic scenes by adapting the dilation scale using the ratio of Gaussian scales before and after temporal changes, unlike Mip-Splatting's fixed dilation scale. Choose for dynamic 4D scenes.

### [Adaptive-Scale Color Offset](13019-adaptive-scale-color-offset.md)
A training-stage module that corrects scale-dependent color mismatch by applying a learnable, scale-specific RGB offset to the SH DC coefficient. Choose when color fidelity across scales is critical and training modification is acceptable.

## Operating characteristics
*   **Time Complexity:** Adds a constant-time filter application per Gaussian during rendering. Integration within pixels may increase per-pixel cost slightly compared to vanilla splatting [S2, unofficial].
*   **Memory Complexity:** No additional memory required for training data structures as it is training-free [S2, unofficial].
*   **Performance:** Performs comparably with or better than Mip-Splatting in various settings and both bounded and unbounded scenes [S2, unofficial].
*   **Strengths:** Training-free; can be applied to any pre-trained Gaussian splatting field as a plugin; explicitly aware of testing frequency [S2, unofficial].
*   **Limitations:** Requires modification of the rendering pipeline to implement the adaptive filter and integration; does not address view-dependent effects as directly as methods like VDGS [S1].

## Background
SA-GS was introduced to address the limitations of Mip-Splatting, which requires modifying the training procedure and uses 3D smoothing/2D Mip filters that are unaware of testing frequency [S2, unofficial]. It builds on the principle that observing Gaussians at different frequencies leads to mismatches between training and testing scales [S2, unofficial]. The method was proposed by Z. Song et al. [S2, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] SA-GS: Scale-Adaptive Gaussian Splatting for Training-Free Anti-Aliasing, https://core.ac.uk/download/620273869.pdf, fetched 2026-10-28 (unofficial)
