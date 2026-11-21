# Color Transfer

Also called: Reference Image Color Transfer, Style Transfer (Color)

## Overview
Color Transfer is a post-processing technique that improves the color match between final volume renderings and an input style image [S1]. It builds on [Reference Image Guided Colorization](13231-reference-image-guided-colorization.md) by mapping the chromatic properties of a reference image onto a target rendering while preserving the target's structural content [S1]. It differs from sibling methods like [Trained Exposure Application](13959-trained-exposure-application.md) by applying a direct statistical color mapping rather than learned exposure adjustments.

## How it works
**Inputs:** Target image $T$ (final volume rendering), Reference image $R$ (style image) [S1].
**Outputs:** Stylized image $O$ [S1].
**Data Structures:** Images are represented as 2D arrays of pixel values. Each pixel is a vector $(r, g, b)$ [G].

**Steps:**
1. **Color Space Conversion:** Convert $T$ and $R$ from RGB to a color space that separates luminance from chrominance, such as CIELAB [G]. Let $L_T, a_T, b_T$ be the channels of $T$ and $L_R, a_R, b_R$ be the channels of $R$.
2. **Statistical Matching:** Compute the mean ($\mu$) and standard deviation ($\sigma$) for the chrominance channels ($a$ and $b$) of both images [G].
   - For the target: $\mu_{a_T}, \sigma_{a_T}, \mu_{b_T}, \sigma_{b_T}$
   - For the reference: $\mu_{a_R}, \sigma_{a_R}, \mu_{b_R}, \sigma_{b_R}$
3. **Normalization and Scaling:** Adjust the chrominance channels of the target image to match the reference's statistics [G].
   - $a'_T = \frac{a_T - \mu_{a_T}}{\sigma_{a_T}} \sigma_{a_R} + \mu_{a_R}$
   - $b'_T = \frac{b_T - \mu_{b_T}}{\sigma_{b_T}} \sigma_{b_R} + \mu_{b_R}$
4. **Reconstruction:** Combine the original luminance channel $L_T$ with the scaled chrominance channels $a'_T$ and $b'_T$ [G].
5. **Conversion:** Convert the resulting Lab image back to RGB to produce $O$ [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def color_transfer_statistical(target_rgb, reference_rgb):
    """
    Transfers color statistics from reference to target in Lab color space.
    Args:
        target_rgb: numpy array of shape (H, W, 3)
        reference_rgb: numpy array of shape (H, W, 3)
    Returns:
        output_rgb: numpy array of shape (H, W, 3)
    """
    import numpy as np
    
    # Convert RGB to Lab color space [G] Standard sRGB to XYZ to Lab matrix conversion
    target_lab = rgb2lab(target_rgb)
    reference_lab = rgb2lab(reference_rgb)
    
    # Extract luminance (L) and chrominance (a, b) channels
    L_T = target_lab[:, :, 0]
    a_T = target_lab[:, :, 1]
    b_T = target_lab[:, :, 2]
    
    a_R = reference_lab[:, :, 1]
    b_R = reference_lab[:, :, 2]
    
    # Compute mean and standard deviation for chrominance channels
    mu_a_T, sigma_a_T = np.mean(a_T), np.std(a_T)
    mu_b_T, sigma_b_T = np.mean(b_T), np.std(b_T)
    mu_a_R, sigma_a_R = np.mean(a_R), np.std(a_R)
    mu_b_R, sigma_b_R = np.mean(b_R), np.std(b_R)
    
    # Guard against zero variance in target chrominance channels
    eps = 1e-8
    sigma_a_T = max(sigma_a_T, eps)
    sigma_b_T = max(sigma_b_T, eps)
    
    # Normalize and scale chrominance channels to match reference statistics
    a_prime = ((a_T - mu_a_T) / sigma_a_T) * sigma_a_R + mu_a_R
    b_prime = ((b_T - mu_b_T) / sigma_b_T) * sigma_b_R + mu_b_R
    
    # Reconstruct image with original luminance and scaled chrominance
    output_lab = np.stack([L_T, a_prime, b_prime], axis=-1)
    
    # Convert back to RGB [G] Standard XYZ to sRGB matrix conversion
    output_rgb = lab2rgb(output_lab)
    return output_rgb

def rgb2lab(rgb):
    """Standard RGB to Lab conversion."""
    # [G] Uses standard sRGB to XYZ to Lab matrices
    pass

def lab2rgb(lab):
    """Standard Lab to RGB conversion."""
    # [G] Uses standard XYZ to sRGB matrices
    pass
```

**Parameters:**
- Color space: Typically CIELAB [G].
- Channel selection: Only chrominance channels are transferred; luminance is preserved from the target [G].

**Edge Cases and Pitfalls:**
- **Degenerate Input:** If the target image is purely grayscale with zero variance in chrominance channels, $\sigma_{a_T}$ or $\sigma_{b_T}$ may be zero, causing division by zero. A small epsilon ($\epsilon = 10^{-8}$) should be added to the denominator [G].
- **Gamut Clipping:** The method assumes the reference image is representative of the desired style. If the reference contains colors outside the gamut of the target's luminance, clipping may occur during RGB conversion [G].

## Subtopics

### [Adaptive Instance Normalization (AdaIN)](14826-adaptive-instance-normalization-adain.md)
Adaptive Instance Normalization (AdaIN) is a feature modulation technique that achieves arbitrary style transfer by aligning the statistical moments (mean and variance) of content feature maps with those of style feature maps. It differs from its parent, Color Transfer, which operates on final pixel colors, by operating in a deep feature space to preserve high-level semantic structure while transferring artistic appearance. Unlike Neural Style Transfer (NST), which optimizes an output image to minimize perceptual loss, AdaIN is a feedforward operation that directly transforms features, enabling rapid, arbitrary style application without per-image optimization.

### PhotoWCT
PhotoWCT is a method that employs whitening and coloring transforms (WCT) to align the feature statistics of the content and style images, aiming to generate consistent and visually coherent results [S2].

## Operating characteristics
Color Transfer is computationally inexpensive, relying on linear statistical operations per channel [G]. It preserves the structural fidelity of the original volume rendering, as it applies no geometric or textural modifications [S1]. However, it is limited to linear statistical matching and cannot transfer complex textures or non-linear artistic styles [G]. It is best used when the primary goal is chromatic consistency with a reference image rather than full artistic stylization [S1].

## Background
Color Transfer was introduced as a practical component in the context of Artistic Radiance Fields (ARF) to improve the color match between final renderings and an input style image [S1]. In ARF, volumetric radiance field rendering consumes significant memory, often limiting optimization to sparsely sampled pixels, which prevents the computation of VGG-based style losses on full-resolution images [S1]. To solve this, ARF employs deferred back-propagation for memory-efficient auto-differentiation with image losses [S1]. Color Transfer is integrated alongside feature matching and VGG-based content/style losses to balance stylization and content preservation [S1].

## Sources
[S1] ARF: Artistic Radiance Fields, https://core.ac.uk/works/391332776, fetched 2026-10-30
[S2] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
