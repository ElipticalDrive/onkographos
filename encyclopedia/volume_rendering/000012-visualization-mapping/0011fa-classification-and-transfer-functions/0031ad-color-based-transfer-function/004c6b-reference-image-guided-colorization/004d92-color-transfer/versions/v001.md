# Color Transfer

Also called: Reference Image Color Transfer, Style Transfer (Color)

## Overview
Color Transfer is a post-processing technique applied to improve the color match between final volume renderings and an input style image [S1]. It serves as a lightweight alternative to full neural style transfer by operating directly on pixel colors rather than deep feature spaces, ensuring the structural content of the target volume remains unchanged while adopting the chromatic properties of the reference [S1].

## How it works
Color Transfer operates on the final rendered image rather than the volumetric data itself. It assumes the existence of a target monochromatic or grayscale rendering $T$ (derived from the volume data) and a reference style image $R$ [S1]. The method maps the color distribution of $R$ onto $T$ to produce a stylized output $O$.

**Inputs:**
- Target image $T$: The grayscale or luminance-based rendering of the volume.
- Reference image $R$: The color image providing the desired style.

**Outputs:**
- Stylized image $O$: The final colorized volume rendering.

**Data Structures:**
- Images are represented as 2D arrays of pixel values. For standard RGB images, each pixel is a vector $(r, g, b)$.

**Steps:**
1. **Color Space Conversion**: Convert $T$ and $R$ from RGB to a color space that separates luminance from chrominance, such as Lab or YCbCr. Let $L_T, a_T, b_T$ be the channels of $T$ and $L_R, a_R, b_R$ be the channels of $R$ [S1].
2. **Statistical Matching**: Compute the mean ($\mu$) and standard deviation ($\sigma$) for the chrominance channels ($a$ and $b$) of both images.
   - For the target: $\mu_{a_T}, \sigma_{a_T}, \mu_{b_T}, \sigma_{b_T}$
   - For the reference: $\mu_{a_R}, \sigma_{a_R}, \mu_{b_R}, \sigma_{b_R}$
3. **Normalization and Scaling**: Adjust the chrominance channels of the target image to match the reference's statistics.
   - $a'_T = \frac{a_T - \mu_{a_T}}{\sigma_{a_T}} \sigma_{a_R} + \mu_{a_R}$
   - $b'_T = \frac{b_T - \mu_{b_T}}{\sigma_{b_T}} \sigma_{b_R} + \mu_{b_R}$
4. **Reconstruction**: Combine the original luminance channel $L_T$ with the scaled chrominance channels $a'_T$ and $b'_T$.
5. **Conversion**: Convert the resulting Lab image back to RGB to produce $O$.

**Parameters:**
- Color space: Typically Lab or YCbCr. Not given in the sources.
- Channel selection: Only chrominance channels are transferred; luminance is preserved from the target. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Degenerate Input**: If the target image is purely grayscale with zero variance in chrominance channels, $\sigma_{a_T}$ or $\sigma_{b_T}$ may be zero, causing division by zero. A small epsilon should be added to the denominator [G].
- **Boundary Conditions**: The method assumes the reference image is representative of the desired style. If the reference contains colors outside the gamut of the target's luminance, clipping may occur during RGB conversion [G].

## Subtopics

### [Adaptive Instance Normalization (AdaIN)](14826-adaptive-instance-normalization-adain.md)
Adaptive Instance Normalization (AdaIN) is a technique for arbitrary style transfer that aligns the statistical moments of feature maps from a content image with those of a style image. It differs from its parent, Color Transfer, which operates on final pixel colors, by operating in a deep feature space to preserve high-level semantic structure while transferring artistic style. Unlike Neural Style Transfer (NST), which optimizes an output image iteratively to minimize perceptual loss, AdaIN is a feedforward operation that directly transforms the content features.

### PhotoWCT
PhotoWCT is a method that employs whitening and coloring transforms (WCT) to align the feature statistics of the content and style images, aiming to generate consistent and visually coherent results [S2].

## Operating characteristics
Color Transfer is computationally inexpensive, requiring only statistical operations on pixel channels [S1]. It preserves the structural fidelity of the original volume rendering completely, as no geometric or textural changes are made [S1]. However, it lacks the ability to transfer complex textures or artistic styles that require non-linear mappings, as it relies on linear statistical matching [S1]. It is best used when the primary goal is chromatic consistency with a reference image rather than artistic stylization.

## Background
Color Transfer was introduced as a practical component in the context of Artistic Radiance Fields (ARF) to address the memory constraints of training neural radiance fields [S1]. The authors noted that volumetric radiance field rendering consumes significant memory, often limiting optimization to sparsely sampled pixels, which prevents the computation of VGG-based style losses on full-resolution images [S1]. To solve this, they devised "deferred back-propagation" to allow memory-efficient auto-differentiation with image losses computed on full-resolution images [S1]. Color Transfer was integrated as a simple technique to improve color match between final renderings and the input style, balancing stylization and content preservation [S1].

## Sources
[S1] ARF: Artistic Radiance Fields, https://core.ac.uk/works/391332776, fetched 2026-10-30
[S2] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
