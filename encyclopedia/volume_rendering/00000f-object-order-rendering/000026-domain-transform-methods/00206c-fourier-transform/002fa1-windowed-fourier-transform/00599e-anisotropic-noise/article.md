# Anisotropic Noise

Also called: Goldberg et al. noise, Oriented Band Filtering, Oriented subband noise

## Overview
Anisotropic noise is a procedural noise function that supports high-quality anisotropic filtering by tiling the frequency domain into oriented subbands [S1, S2]. Unlike isotropic noise functions (such as [Perlin noise](10802-perlin-noise.md)) which force a trade-off between aliasing artifacts and loss of detail on tilted surfaces, anisotropic noise adapts its frequency content to non-uniform perspective distortions by blending pre-computed oriented bands [S1, S2]. It sets itself apart from its sibling [Aperture Apodization](10803-aperture-apodization.md) by explicitly modeling orientation in the frequency decomposition, allowing it to approximate any desired anisotropic frequency spectrum on a surface [S1, S2].

## How it works
**Inputs:**
- A set of pre-computed oriented noise subband textures.
- The number of bands $B$ (typically 4 or 8).
- The target anisotropic filter characteristics (defined by the surface geometry and view direction).

**Outputs:**
- A scalar noise value $N(x, y)$ for a given 2D coordinate $(x, y)$ on the surface.

**Data Structures:**
- **Subband Textures:** The oriented subbands are stored in 32-bit RGBA images. Each channel (R, G, B, A) stores one oriented subband.
  - One texture stores 4 orientations (e.g., 0°, 45°, 90°, 135°).
  - Two textures store 8 orientations.
- **Scale:** Subbands are precomputed at a single scale only. Other scales are generated on the fly by scaling the texture coordinates.

**Method Steps:**

1.  **Preprocessing (Offline):**
    1.  Partition the frequency domain using steerable filters to create oriented subbands tightly localized in scale and orientation [S1, S2].
    2.  Synthesize noise tiles for each subband. This is typically done by filtering white noise with the steerable filters [S1, S2].
    3.  Pack each oriented subband image into one channel of a 32-bit RGBA image, yielding four orientations per texture [S1, S2].
    4.  Store the textures. Note that the filters implement an invertible transform, allowing exact recovery of a signal from its decomposition [S2].

2.  **Runtime (Per Pixel):**
    1.  Determine the appropriate weights for each oriented subband based on the local anisotropic filter requirements (e.g., derived from the Jacobian of the surface mapping) [S1, S2].
    2.  Sample the pre-computed subband textures at the current pixel location $(x, y)$. Since subbands are precomputed at a single scale, scale the texture coordinates to match the required frequency scale [S1, S2].
    3.  Compute the final noise value as a weighted sum of the sampled noise subbands:
        $$ N(x, y) = \sum_{i=1}^{B} w_i \cdot S_i(x, y) $$
        where $w_i$ is the weight for the $i$-th oriented subband, and $S_i(x, y)$ is the value of the $i$-th subband at $(x, y)$ [S1, S2].
    4.  The weights $w_i$ are adapted with respect to the anisotropic filter to allow noise content to adapt to non-uniform perspective distortions [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def anisotropic_noise(x, y, weights, subband_textures):
    """
    Compute anisotropic noise value at (x, y).
    
    Args:
        x, y: 2D coordinates on the surface.
        weights: List of B weights [w_0, ..., w_{B-1}] for the oriented subbands.
        subband_textures: List of B textures (or channels thereof) containing the pre-computed subbands.
                          Each texture is sampled at the appropriate scale.
    
    Returns:
        Scalar noise value.
    """
    noise_value = 0.0
    
    # B is the number of bands (e.g., 4 or 8)
    B = len(weights)
    
    for i in range(B):
        # Sample the i-th subband texture at (x, y)
        # Note: In practice, texture coordinates may need scaling for different frequencies,
        # but the core method uses a single precomputed scale per band, scaled via UVs.
        # The source states: "All other scales are generated on the fly by simply scaling the precomputed textures."
        subband_value = sample_texture(subband_textures[i], x, y)
        
        # Weighted sum
        noise_value += weights[i] * subband_value
        
    return noise_value
```

**Parameters:**
- **Number of Bands ($B$):** Typically 4 or 8. Using 4 bands (1 texture) or 8 bands (2 textures) leads to a good trade-off between storage, rendering speed, and image quality [S1, S2].
- **Scale:** Precomputed at a single scale. Other scales are handled by texture coordinate scaling [S1, S2].

**Edge Cases and Pitfalls:**
- **Precision:** The method relies on linear interpolation of steerable filters to avoid interpolation artifacts when blending subbands [S1, S2].
- **Boundaries:** The invertible transform property ensures exact recovery of signals, but this assumes the decomposition is complete [S2].
- **Degenerate Input:** If the anisotropic filter becomes isotropic, the weights should converge to equal values for all orientations (or appropriate isotropic combination) [S2].

## Subtopics

### [Fourier Spectral Synthesis](16451-fourier-spectral-synthesis.md)
Fourier spectral synthesis generates a noise function with a specific target power spectrum by filtering white noise in the frequency domain. It sets itself apart from spatial-domain methods (like [Sparse Convolution Noise](16455-sparse-convolution-noise.md)) by enforcing spectral properties directly, making it the foundational technique for [Anisotropic Noise](16450-anisotropic-noise.md) and [Spot Noise](16452-spot-noise.md).

### [Gabor Noise Surface Filtering](16556-gabor-noise-surface-filtering.md)
Gabor Noise Surface Filtering is an analytical method for generating anisotropically filtered procedural noise on surfaces by exploiting the spectral structure of Gabor noise. Unlike methods that discretize the spectrum into bands or apply post-hoc blurring, this approach computes the exact filtered spectrum per pixel by multiplying the Gabor kernel's frequency-domain Gaussian with the view-dependent filter Gaussian, then generating noise from the resulting parameters. It is distinct from [Fourier Spectral Synthesis](16451-fourier-spectral-synthesis.md) because it operates on oriented subbands (Gabor kernels) rather than a global isotropic spectrum, allowing it to adapt to non-uniform perspective distortions and anisotropic filtering requirements without aliasing or over-blurring.

## Operating characteristics
- **Time Complexity:** Generally faster than Perlin noise [S2]. The runtime cost is dominated by texture sampling and a weighted sum, which is $O(B)$ where $B$ is the number of bands [S1, S2].
- **Memory Complexity:** Storage is proportional to the number of bands and the texture resolution. Typically 1-2 textures for 4-8 bands [S1, S2].
- **Measured Performance:** Anisotropic noise is generally faster than Perlin noise [S2].
- **Strengths:** Supports high-quality anisotropic filtering, solid noise, and isotropic filtering [S2]. Adapts to non-uniform perspective distortions [S1].
- **Limitations:** Requires offline preprocessing to generate subbands [S1, S2]. Limited to the orientations present in the pre-computed bands (though steerable filters allow interpolation) [S1, S2].
- **When to use:** When high-quality anisotropic filtering is required on surfaces with non-uniform perspective distortions [S1, S2].
- **When not to use:** When memory is extremely constrained and pre-computed textures are not feasible, or when isotropic filtering is sufficient and simpler methods like Perlin noise are preferred for their ubiquity [S2].

## Background
Anisotropic noise was introduced by Goldberg et al. in 2008 [GZD08] [S1, S2]. It was developed to address the limitations of existing noise functions which only supported isotropic filtering, leading to a trade-off between aliasing artifacts and loss of detail on tilted surfaces [S1, S2]. The method is based on steerable filters [SF95, PS00] that partition the frequency domain [S1, S2]. It is used for surface noise via 2D texture mapping and compensating for parametric distortion, as well as for anisotropic analytic filtering [S1, S2].

## Sources
[S1] PDF A Survey of Procedural Noise Functions, https://www.cs.umd.edu/~zwicker/publications/SurveyProceduralNoise-CGF10.pdf, fetched 2026-10-27
[S2] PDF State of the Art in Procedural Noise Functions, https://www-sop.inria.fr/reves/Basilic/2010/LLCDDELPZ10/LLCDDELPZ10STARPNF.pdf, fetched 2026-10-27
