# Anisotropic Noise

Also called: Goldberg et al. noise, Oriented Band Filtering, Oriented subband noise

## Overview
Anisotropic noise is a procedural noise function that supports high-quality anisotropic filtering by tiling the frequency domain into oriented subbands [S1, S2]. Unlike isotropic noise functions which force a trade-off between aliasing artifacts and loss of detail on tilted surfaces, anisotropic noise adapts its frequency content to non-uniform perspective distortions by blending pre-computed oriented bands [S1, S2]. It sets itself apart from its sibling [Aperture Apodization](10803-aperture-apodization.md) by explicitly modeling orientation in the frequency decomposition, allowing it to approximate any desired anisotropic frequency spectrum on a surface [S1, S2].

## How it works
**Inputs:**
- A set of pre-computed oriented noise subband textures [S2].
- The number of bands $B$ (typically 4 or 8) [S1, S2].
- The target anisotropic filter characteristics (defined by the surface geometry and view direction) [S1].

**Outputs:**
- A scalar noise value $N(x, y)$ for a given 2D coordinate $(x, y)$ on the surface [S1].

**Data Structures:**
- **Subband Textures:** The oriented subbands are stored in 32-bit RGBA images. Each channel (R, G, B, A) stores one oriented subband [S2]. One texture stores 4 orientations [S2]. Two textures store 8 orientations [S2].
- **Scale:** Subbands are precomputed at a single scale only. Other scales are generated on the fly by scaling the texture coordinates [S1, S2].

**Method Steps:**
1. **Preprocessing (Offline):**
   1. Partition the frequency domain using steerable filters to create oriented subbands tightly localized in scale and orientation [S1, S2].
   2. Synthesize noise tiles for each subband by filtering white noise with the steerable filters [S1, S2].
   3. Pack each oriented subband image into one channel of a 32-bit RGBA image, yielding four orientations per texture [S1, S2].
   4. Store the textures. The filters implement an invertible transform, allowing exact recovery of a signal from its decomposition [S2].
2. **Runtime (Per Pixel):**
   1. Determine the appropriate weights for each oriented subband based on the local anisotropic filter requirements [S1, S2].
   2. Scale the texture coordinates to match the required frequency scale, since subbands are precomputed at a single scale [S1, S2].
   3. Sample the pre-computed subband textures at the scaled pixel location $(x, y)$ [S1, S2].
   4. Compute the final noise value as a weighted sum of the sampled noise subbands:
      $$ N(x, y) = \sum_{i=1}^{B} w_i \cdot S_i(x, y) $$
      where $w_i$ is the weight for the $i$-th oriented subband, and $S_i(x, y)$ is the value of the $i$-th subband at $(x, y)$ [S1, S2].
   5. Adapt the weights $w_i$ with respect to the anisotropic filter to allow noise content to adapt to non-uniform perspective distortions [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_texture(tex, u, v, width, height):
    """Bilinear sample a 2D texture array with wrapping."""
    u = u - int(u)
    v = v - int(v)
    x = u * (width - 1)
    y = v * (height - 1)
    x0, y0 = int(x), int(y)
    x1, y1 = min(x0 + 1, width - 1), min(y0 + 1, height - 1)
    dx, dy = x - x0, y - y0
    val = (1 - dx) * (1 - dy) * tex[y0][x0] + \
          dx * (1 - dy) * tex[y0][x1] + \
          (1 - dx) * dy * tex[y1][x0] + \
          dx * dy * tex[y1][x1]
    return val

def anisotropic_noise(x, y, scale, weights, subband_textures, tex_width, tex_height):
    """
    Compute anisotropic noise value at (x, y).
    
    Args:
        x, y: 2D coordinates on the surface.
        scale: Scaling factor for texture coordinates to adjust frequency.
        weights: List of B weights [w_0, ..., w_{B-1}] for the oriented subbands.
        subband_textures: List of B 2D arrays containing the pre-computed subbands.
        tex_width, tex_height: Dimensions of the subband textures.
    
    Returns:
        Scalar noise value.
    """
    noise_value = 0.0
    B = len(weights)
    scaled_x = x * scale
    scaled_y = y * scale
    
    for i in range(B):
        subband_value = sample_texture(subband_textures[i], scaled_x, scaled_y, tex_width, tex_height)
        noise_value += weights[i] * subband_value
        
    return noise_value
```

**Parameters:**
- **Number of Bands ($B$):** Typically 4 or 8. Using 4 bands (1 texture) or 8 bands (2 textures) leads to a good trade-off between storage, rendering speed, and image quality [S1, S2].
- **Scale:** Precomputed at a single scale. Other scales are handled by scaling the texture coordinates [S1, S2].

**Edge Cases and Pitfalls:**
- **Precision:** The method relies on linear interpolation of steerable filters to avoid interpolation artifacts when blending subbands [S1, S2].
- **Boundaries:** The invertible transform property ensures exact recovery of signals, but this assumes the decomposition is complete [S2].
- **Degenerate Input:** If the anisotropic filter becomes isotropic, the weights adapt accordingly to approximate the isotropic spectrum [S1].

## Subtopics

### [Fourier Spectral Synthesis](16451-fourier-spectral-synthesis.md)
Fourier spectral synthesis generates a noise function with a specific target power spectrum by filtering white noise in the frequency domain. It sets itself apart from spatial-domain methods (like [Sparse Convolution Noise](16455-sparse-convolution-noise.md)) by enforcing spectral properties directly, making it the foundational technique for [Anisotropic Noise](16450-anisotropic-noise.md) and [Spot Noise](16452-spot-noise.md).

### [Gabor Noise Surface Filtering](16556-gabor-noise-surface-filtering.md)
Gabor Noise Surface Filtering is an analytical method for generating anisotropically filtered procedural noise on surfaces, solving the problem of aliasing and visual inconsistency caused by non-uniform perspective distortions during camera or object motion. Unlike methods that discretize the spectrum into distinct bands or rely on post-hoc blurring, this approach exploits the fact that Gabor kernels are Gaussians in the frequency domain, allowing the filter to be applied by analytically modifying the kernel parameters per pixel. It builds on [Anisotropic Noise](16450-anisotropic-noise.md) and provides a view-dependent spectral control that adapts to the screen-space projection of the surface.

## Operating characteristics
- **Time Complexity:** Generally faster than Perlin noise [S3]. The runtime cost is dominated by texture sampling and a weighted sum, which is $O(B)$ where $B$ is the number of bands [S1, S2].
- **Memory Complexity:** Storage is proportional to the number of bands and the texture resolution. Typically 1-2 textures for 4-8 bands [S1, S2].
- **Measured Performance:** Anisotropic noise is generally faster than Perlin noise [S3].
- **Strengths:** Supports high-quality anisotropic filtering, solid noise, and isotropic filtering [S3]. Adapts to non-uniform perspective distortions [S1].
- **Limitations:** Requires offline preprocessing to generate subbands [S1, S2]. Limited to the orientations present in the pre-computed bands, though steerable filters allow interpolation [S1, S2].
- **When to use:** When high-quality anisotropic filtering is required on surfaces with non-uniform perspective distortions [S1, S2].
- **When not to use:** When memory is extremely constrained and pre-computed textures are not feasible [S1, S2].

## Background
Anisotropic noise was introduced by Goldberg et al. in 2008 [GZD08] [S1, S2]. It was developed to address the limitations of existing noise functions which only supported isotropic filtering, leading to a trade-off between aliasing artifacts and loss of detail on tilted surfaces [S1, S2]. The method is based on steerable filters [SF95, PS00] that partition the frequency domain [S1, S2]. It is used for surface noise via 2D texture mapping and compensating for parametric distortion, as well as for anisotropic analytic filtering [S1, S2].

## Sources
[S1] A Survey of Procedural Noise Functions, https://doi.org/10.1111/j.1467-8659.2010.01827.x, fetched 2026-10-27
[S2] PDF A Survey of Procedural Noise Functions, https://www.cs.umd.edu/~zwicker/publications/SurveyProceduralNoise-CGF10.pdf, fetched 2026-10-27
[S3] PDF State of the Art in Procedural Noise Functions, https://www-sop.inria.fr/reves/Basilic/2010/LLCDDELPZ10/LLCDDELPZ10STARPNF.pdf, fetched 2026-10-27
