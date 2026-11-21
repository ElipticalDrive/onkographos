# Bloom
Also called: Bloom Post-Processing, Cross-shaped bloom, Luminance-based bright pass, Multi-Scale Gaussian Bloom

## Overview
Bloom is a screen-space post-processing effect that enhances the perception of brightness by isolating luminous regions via a soft threshold, blurring them across a pyramid of scales, and adding the result back to the original image [S1, S2, unofficial]. It differs from direct compositing methods by operating entirely in screen space on the final display image rather than during ray traversal [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A display image (e.g., `.png`, `.ppm`) representing the rendered scene [S1, unofficial].
- Output: An image of the same resolution with the bloom effect applied [S1, unofficial].

**Data Structures:**
- The method operates on a 2D array of luminance values [G].

**Steps:**
1. **Luminance-based Bright Pass:** Identify bright parts of the image using a soft threshold [S2, unofficial]. Pixels below the threshold contribute minimally, while those above contribute proportionally to their excess brightness [S2, unofficial].
2. **Multi-Scale Gaussian Blur:** Apply a Gaussian blur to the bright pass over a pyramid of scales [S1, S2, unofficial]. Multiple blurred versions are generated at different kernel sizes or resolutions [S1, unofficial].
3. **Accumulation:** Add the blurred result back to the original image [S1, unofficial].

**Formulas:**
Let $I(x,y)$ be the original image luminance at pixel $(x,y)$ [G].
Let $T$ be the soft threshold for the bright pass [G].
Let $B(x,y) = \max(0, I(x,y) - T)$ be the bright pass mask [G].
Let $G_k$ be a Gaussian kernel with scale $k$ [G].
The blurred bright pass at scale $k$ is computed as $B_k(x,y) = B(x,y) * G_k(x,y)$ [G].
The total bloom contribution $W(x,y)$ is the sum over all scales: $W(x,y) = \sum_k B_k(x,y)$ [G].
The final image is $I_{final}(x,y) = I(x,y) + W(x,y)$ [G].

**Parameters:**
- Threshold $T$: Not given in the sources. [G]
- Gaussian scales: Not given in the sources. [G]
- Number of scales: Not given in the sources. [G]

**Edge Cases and Pitfalls:**
- Precision: High-intensity values may overflow if the addition exceeds the bit depth of the target format [G].
- Boundaries: The blur operation requires handling image boundaries, typically by clamping or mirroring pixel values [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def apply_bloom(image, threshold, scales):
    """
    Apply bloom effect to an image.
    Args:
        image: 2D array of luminance values (floats).
        threshold: Soft threshold for bright pass.
        scales: List of Gaussian kernel standard deviations.
    Returns:
        Bloomed image.
    """
    H, W = len(image), len(image[0])
    # Step 1: Bright Pass
    bright_pass = [[max(0.0, image[r][c] - threshold) for c in range(W)] for r in range(H)]
    
    # Step 2: Multi-Scale Gaussian Blur
    blurred_bloom = [[0.0] * W for _ in range(H)]
    for sigma in scales:
        R = int(3 * sigma) + 1
        kernel = [[math.exp(-(i**2 + j**2) / (2 * sigma**2)) for j in range(-R, R+1)] for i in range(-R, R+1)]
        kernel_sum = sum(sum(row) for row in kernel)
        kernel = [[v / kernel_sum for v in row] for row in kernel]
        
        blurred_scale = [[0.0] * W for _ in range(H)]
        for r in range(H):
            for c in range(W):
                acc = 0.0
                for ki in range(-R, R+1):
                    for kj in range(-R, R+1):
                        nr, nc = r + ki, c + kj
                        # Clamp boundaries
                        nr = max(0, min(H-1, nr))
                        nc = max(0, min(W-1, nc))
                        acc += bright_pass[nr][nc] * kernel[ki+R][kj+R]
                blurred_scale[r][c] = acc
        blurred_bloom = [[blurred_bloom[r][c] + blurred_scale[r][c] for c in range(W)] for r in range(H)]
        
    # Step 3: Accumulation
    final_image = [[image[r][c] + blurred_bloom[r][c] for c in range(W)] for r in range(H)]
    return final_image
```

## Operating characteristics
- **Time Complexity:** Not given in the sources. [G]
- **Memory Complexity:** Not given in the sources. [G]
- **Strengths:** Simple to implement, effective at enhancing perceived brightness and glow [G].
- **Limitations:** Can introduce halo artifacts if the threshold or blur radius is not chosen carefully [G].
- **When to use:** When a glow effect is desired for bright light sources [G].
- **When not to use:** When performance is critical and the effect is not needed, or when precise control over light diffusion is required beyond simple blurring [G].

## Background
Bloom is used in screen-space post-processing pipelines for display images [S1, S2, unofficial].

## Sources
[S1] Milestone 13: MagicaVoxel look (.vox, DDA voxel grid, sun/sky, bloom, style voxel), https://github.com/adriendjn/mypbr/pull/10, fetched 2026-10-25 (unofficial)
[S2] Add volumetric rendering, advanced materials, and imager effects, https://github.com/MihrimatriX/photon-engine/pull/3, fetched 2026-10-28 (unofficial)
