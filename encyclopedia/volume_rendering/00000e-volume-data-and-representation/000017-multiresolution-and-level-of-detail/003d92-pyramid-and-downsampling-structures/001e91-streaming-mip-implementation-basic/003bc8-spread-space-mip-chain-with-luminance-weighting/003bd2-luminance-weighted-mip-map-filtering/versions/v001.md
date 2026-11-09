# Luminance Weighted MIP Map Filtering

Also called: Masking idea

## Overview
Luminance Weighted MIP Map Filtering is a technique to reduce illumination leaking artifacts in Gaussian MIP map blurring by using an auxiliary mask $M$ to prevent low-standard-deviation pixels from propagating to high MIP levels [S1, unofficial]. It differs from its parent [Spread-space MIP chain with luminance weighting](9093-spread-space-mip-chain-with-luminance-weighting.md) by adding a luminance-weighted mask to prevent illumination leaking artifacts that can occur when naively applying this blur to high-contrast scenes.

## How it works
**Inputs:**
- $\mathbb{L}_{\text{sc}}^{[0]}$: Scattered radiance at base MIP level.
- $\mathbb{W}^{[0]}$: Spreading width (or related luminance metric) at base level.
- $T$: Masking threshold distance.
- $\varepsilon$: Width control parameter for the smoothstep transition.
- $\mathcal{G}$: Discrete Gaussian filter kernel.

**Outputs:**
- $\mathbb{L}_{\text{sc}}^{[k]}$: Filtered radiance at MIP level $k$.
- $\mathbb{M}^{[k]}$: Auxiliary mask at MIP level $k$.

**Data Structures:**
- $\mathbb{L}_{\text{sc}}^{[k]}$: 2D texture/array of radiance values (e.g., RGB or scalar).
- $\mathbb{W}^{[k]}$: 2D texture/array of spreading width/luminance values.
- $\mathbb{M}^{[k]}$: 2D texture/array of mask values in range $[0, 1]$.

**Steps:**
1. Initialize base level $k=0$ with input radiance $\mathbb{L}_{\text{sc}}^{[0]}$ and width $\mathbb{W}^{[0]}$.
2. For each subsequent MIP level $k > 0$:
   a. Compute the mask $\mathbb{M}^{[k]}$ using the width from the previous level $\mathbb{W}^{[k-1]}$:
      $$ \mathbb{M}^{[k]} = \text{smoothstep}\left(T, (1 + \varepsilon) \cdot T, \mathbb{W}^{[k-1]}\right) $$
      where $\text{smoothstep}(e_1, e_2, x)$ returns $0$ for $x \le e_1$, $1$ for $x \ge e_2$, and interpolates smoothly between $e_1$ and $e_2$ otherwise [S1, unofficial].
   b. Apply the mask to the previous level's radiance:
      $$ \mathbb{L}_{\text{masked}} = \mathbb{M}^{[k]} \cdot \mathbb{L}_{\text{sc}}^{[k-1]} $$
   c. Convolve the masked radiance with the Gaussian filter $\mathcal{G}$ to produce the current level:
      $$ \mathbb{L}_{\text{sc}}^{[k]} = \mathbb{L}_{\text{masked}} * \mathcal{G} $$
      (Equation 4.19 in [S1, unofficial])

**Parameters:**
- $T$: Masking threshold. Controls the distance/width below which pixels are suppressed. Typical values depend on scene scale.
- $\varepsilon$: Controls the width of the transition region in the smoothstep. Small positive values create a sharp cutoff; larger values create a softer transition.

**Edge Cases and Pitfalls:**
- If $\mathbb{W}^{[k-1]}$ is uniformly below $T$, the mask becomes zero, and the MIP level becomes black. This is intended behavior to prevent leaking.
- Precision issues may arise if $T$ is very small or $\varepsilon$ is extremely small, leading to binary masking rather than smooth weighting.
- The definition of $\mathbb{W}$ (spreading width) must be consistent with the parent method [Spread-space MIP chain with luminance weighting](9093-spread-space-mip-chain-with-luminance-weighting.md).

## Subtopics

## Operating characteristics
- **Time Complexity:** Building each MIP level requires one mask computation (pixel-wise smoothstep) and one Gaussian convolution. The convolution cost depends on kernel size. Overall, linear in the number of pixels per level.
- **Memory Complexity:** Requires storing an additional mask texture $\mathbb{M}^{[k]}$ at each level, or recomputing it on the fly if memory is constrained.
- **Strengths:** Effectively prevents illumination leaking from bright, small objects (e.g., lanterns) into dark, large backgrounds, which is a common artifact in naive MIP map filtering [S1, unofficial].
- **Limitations:** Requires an auxiliary mask and an additional width/luminance texture. The choice of $T$ and $\varepsilon$ can affect visual quality and may require tuning.
- **When to use:** When using Gaussian MIP map blurring for real-time light transport or scattering, and illumination leaking artifacts are visible.
- **When not to use:** If the scene has uniform luminance or if the cost of maintaining the mask is prohibitive.

## Background
Originated in the context of real-time light transport in analytically integrable participating media. The technique is attributed to Elek et al. [2013] and is described in the thesis "Real-Time Light Transport in Analytically Integrable Participating Media" [S1, unofficial]. It addresses the issue of sharp discontinuities and incorrect light leaking seen in naive MIP map filtering approaches.

## Sources
[S1] Real-Time Light Transport in Analytically Integrable Participating Media, https://cgg.mff.cuni.cz/~tomas/publications/pub-thesis-real-time-scattering/thesis.pdf, fetched 2026-10-29 (unofficial)
