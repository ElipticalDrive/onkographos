# Dynamic Texture Slice Resampling

Also called: Adaptive Patterns, Adaptive pattern evaluation, Adaptive stripe pattern, Dynamic resampling

## Overview
Dynamic Texture Slice Resampling is a texturing technique that adjusts sampling frequency to yield approximately constant image-space pattern density, compensating for highly anisotropic stretching of integral surface parameterization [S1]. It solves the problem of visual artifacts (gaps, obscured flow) caused by strong surface distortion where natural parameterization stretches the surface by factors of 1000 or greater [S1]. Unlike [Gradient-driven Sampling](5281-gradient-driven-sampling.md) which adapts step size based on data gradients, or [Dynamic Adaptive Sampling Ray-Casting](7687-dynamic-adaptive-sampling-ray-casting.md) which adapts to volume density, this method adapts texture frequency to maintain visual consistency of surface features (like stripes) regardless of geometric distortion [S1, S2].

## How it works
**Inputs:**
*   Texture coordinates $(s, t)$ for the current pixel.
*   Image-space partial derivatives of texture coordinates: $\frac{\partial s}{\partial i}, \frac{\partial s}{\partial j}, \frac{\partial t}{\partial i}, \frac{\partial t}{\partial j}$ [S1].
*   A base pattern function $P(s, t)$ defined over the unit square [S1].

**Outputs:**
*   Resampled color/value $c(s, t)$ with constant image-space frequency [S1].

**Data Structures:**
*   Integer resolution levels $l_s, l_t$ [S1].
*   Fractional parts $\tilde{l}_s, \tilde{l}_t$ [S1].
*   Pre-computed or procedurally generated pattern instances $\hat{P}_{l_s, l_t}$ [S1].

**Steps:**

1.  **Calculate Image-Space Variation:**
    Compute the variation $\lambda_s$ and $\lambda_t$ in texture coordinates in image space at pixel $(i, j)$ using the image-space partial derivatives [S1]:
    $$ \lambda_s(i, j) = \sqrt{ \left(\frac{\partial s}{\partial i}\right)^2 + \left(\frac{\partial s}{\partial j}\right)^2 } $$
    $$ \lambda_t(i, j) = \sqrt{ \left(\frac{\partial t}{\partial i}\right)^2 + \left(\frac{\partial t}{\partial j}\right)^2 } $$
    [S1]

2.  **Determine Resolution Levels:**
    Calculate the integer resolution levels $l_s$ and $l_t$ via logarithmic scaling [S1]:
    $$ l_s = \log_2(\lambda_s) $$
    $$ l_t = \log_2(\lambda_t) $$
    [S1]

3.  **Compute Fractional Parts:**
    Determine the fractional parts for interpolation [S1]:
    $$ \tilde{l}_s = l_s - \lfloor l_s \rfloor $$
    $$ \tilde{l}_t = l_t - \lfloor l_t \rfloor $$
    [S1]

4.  **Define Frequency-Adjusted Pattern:**
    Define the frequency-adjusted pattern $\hat{P}_{l_s, l_t}$ by evaluating the base pattern $P$ with compensated frequency [S1]:
    $$ \hat{P}_{l_s, l_t}(s, t) := P(s \cdot 2^{-l_s}, t \cdot 2^{-l_t}) $$
    [S1]
    *Note:* If either $\lambda_s$ or $\lambda_t$ doubles, the pattern frequency in the corresponding direction is halved to yield the same image-space frequency [S1].

5.  **Interpolate Pattern:**
    Calculate the final color $c(s, t)$ using bilinear interpolation between neighboring resolution levels to ensure smooth frequency transitions [S1]:
    $$ c(s, t) = (1 - \tilde{l}_s)(1 - \tilde{l}_t)\hat{P}_{\lfloor l_s \rfloor, \lfloor l_t \rfloor}(s, t) + \tilde{l}_t \hat{P}_{\lfloor l_s \rfloor, \lceil l_t \rceil}(s, t) + \tilde{l}_s(1 - \tilde{l}_t)\hat{P}_{\lceil l_s \rceil, \lfloor l_t \rfloor}(s, t) + \tilde{l}_t \hat{P}_{\lceil l_s \rceil, \lceil l_t \rceil}(s, t) $$
    [S1]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dynamic_texture_slice_resampling(s, t, ds_di, ds_dj, dt_di, dt_dj, P):
    """
    s, t: Texture coordinates
    ds_di, ds_dj: Partial derivatives of s
    dt_di, dt_dj: Partial derivatives of t
    P: Function returning pattern value for given (u, v)
    """
    # Step 1: Calculate image-space variation
    lambda_s = sqrt(ds_di**2 + ds_dj**2)
    lambda_t = sqrt(dt_di**2 + dt_dj**2)

    # Step 2: Determine resolution levels
    l_s = log2(lambda_s)
    l_t = log2(lambda_t)

    # Step 3: Compute fractional parts
    l_s_floor = floor(l_s)
    l_t_floor = floor(l_t)
    l_s_ceil = l_s_floor + 1
    l_t_ceil = l_t_floor + 1
    
    l_tilde_s = l_s - l_s_floor
    l_tilde_t = l_t - l_t_floor

    # Step 4 & 5: Define adjusted patterns and interpolate
    # Helper to get pattern at specific resolution level
    def get_pattern(u, v, ls, lt):
        return P(u * (2**(-ls)), v * (2**(-lt)))

    p00 = get_pattern(s, t, l_s_floor, l_t_floor)
    p01 = get_pattern(s, t, l_s_floor, l_t_ceil)
    p10 = get_pattern(s, t, l_s_ceil, l_t_floor)
    p11 = get_pattern(s, t, l_s_ceil, l_t_ceil)

    # Bilinear interpolation
    c = ((1 - l_tilde_s) * (1 - l_tilde_t) * p00 +
         l_tilde_t * p01 +
         l_tilde_s * (1 - l_tilde_t) * p10 +
         l_tilde_t * p11)
    
    return c
```

**Edge Cases and Pitfalls:**
*   **Anisotropic Distortion:** Natural integral surface parameterization can stretch the surface by a factor of 1000 or greater. Without adaptation, this causes large gaps between stripes and obscures flow depiction [S1].
*   **Derivative Calculation:** While Equations 3 and 4 seem difficult to evaluate, they can leverage built-in rendering system functionality (e.g., hardware derivatives) and are cheap to evaluate [S1].
*   **Discrete Levels:** Resolution levels are discretely defined, necessitating interpolation to avoid visible frequency jumps [S1].

## Subtopics

### [Epipolar Slice Distribution](6374-epipolar-slice-distribution.md)
A non-uniform sampling strategy distributing epipolar slices along the image perimeter to guarantee proper sampling density while minimizing total slices. It strictly bounds maximum sampling error to half a pixel without upsampling, achieving tighter efficiency when the epipole is near screen corners.

## Operating characteristics
*   **Complexity:** The method relies on image-space partial derivatives, which are typically available or cheap to compute in modern rendering pipelines [S1]. The pattern evaluation and interpolation are constant-time operations per pixel.
*   **Performance:** Maintains high rendering performance by avoiding complex geometric resampling or non-uniform sampling step adjustments during DVR [S2].
*   **Strengths:** Effectively highlights surface shape, distortion, and flow behavior directly on integral surfaces by maintaining constant image-space pattern density [S1]. Compensates for perspective foreshortening [S1].
*   **Limitations:** Requires access to texture coordinate derivatives. The method is specific to surface texturing (integral surfaces) rather than volumetric density sampling [S1, S2].

## Background
The method was introduced in the context of illustrative rendering for integral surfaces to address the visual degradation of stripe textures under strong anisotropic stretching [S1]. It builds on the concept of repurposing mipmapping-like mechanisms (loading stacks of successively smaller images) but adapts it by reusing a single pattern and adjusting sampling frequency dynamically [S1]. A related concept in hardware-accelerated DVR involves dynamically updating slicing steps according to viewing directions to maintain constant step size and avoid striping artifacts [S2].

## Sources
[S1] IRIS: Illustrative Rendering for Integral Surfaces, https://doi.org/10.1109/tvcg.2010.173, fetched 2026-10-27
[S2] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
