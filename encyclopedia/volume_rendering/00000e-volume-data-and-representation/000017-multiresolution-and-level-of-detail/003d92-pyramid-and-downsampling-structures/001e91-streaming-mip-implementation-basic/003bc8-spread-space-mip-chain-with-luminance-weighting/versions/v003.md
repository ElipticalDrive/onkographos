# Spread-space MIP chain with luminance weighting

Also called: Luminance-weighted MIP chain, Masked MIP chain

## Overview
This method constructs a multi-resolution pyramid for scattering images by combining spatial blurring with a luminance-weighted auxiliary mask to prevent illumination leaking artifacts [S1, unofficial]. It differs from its parent [Streaming MIP Implementation (Basic)](1803-streaming-mip-implementation-basic.md) by introducing a spreading width chain $\mathbf{W}$ and a luminance mask $\mathbf{Y}$ that modulate the Gaussian blur, ensuring that low-standard-deviation (sharp) pixels do not incorrectly propagate brightness to high MIP levels [S1, unofficial]. It differs from [Conical MImP](7656-conical-mimp.md) by using a separable Gaussian blur with explicit masking rather than a conical volume approximation [G].

## How it works
**Inputs:**
- $\mathbf{L}_{\text{sc}}^{[0]}$: The input scattering image at MIP level 0 [S1, unofficial].
- $\mathbf{W}^{[0]}$: The initial spreading width image (standard deviation map) at level 0 [S1, unofficial].
- $K$: The maximum MIP level [S1, unofficial].
- $c$: A constant defining the base threshold distance (from Equation 3.7 of [S1, unofficial]).
- $\varepsilon$: The width parameter for the smoothstep masking function [S1, unofficial].

**Outputs:**
- $\mathbf{L}_{\text{sc}}^{[0...K]}$: The MIP chain of scattering images [S1, unofficial].
- $\mathbf{W}^{[0...K]}$: The MIP chain of spreading widths [S1, unofficial].
- $\mathbf{Y}^{[0...K]}$: The auxiliary luminance weighting mask chain [S1, unofficial].
- $\mathbf{M}^{[1...K]}$: The auxiliary masking chain (computed on-the-fly or stored) [S1, unofficial].

**Data Structures:**
- Images are 2D arrays of floating-point values [G].
- $\mathbf{L}_{\text{sc}}$: RGB color values [S1, unofficial].
- $\mathbf{W}$: Single-channel scalar values representing spreading width [S1, unofficial].
- $\mathbf{Y}$: Single-channel scalar values representing luminance [S1, unofficial].
- $\mathbf{M}$: Single-channel scalar values representing the mask, ranging from 0.0 to 1.0 [S1, unofficial].

**Definitions:**
- $\mathcal{G}$: Discrete Gaussian filter kernel [S1, unofficial].
- $\mathcal{U}$: Uniform distribution convolution kernel (averaging filter) of the same size as $\mathcal{G}$ [S1, unofficial].
- $y(\mathbf{C})$: Function computing absolute RGB luminance from color $\mathbf{C}$ [S1, unofficial]. Standard formula: $y = 0.2126 R + 0.7152 G + 0.0722 B$ [G].
- $\text{smoothstep}(e_1, e_2, x)$: Smooth interpolation function [G]. Returns 0 if $x \le e_1$, 1 if $x \ge e_2$, and smooth interpolation between 0 and 1 if $e_1 < x < e_2$ [G].

**Algorithm Steps:**

1. **Initialize Level 0:**
   Set $\mathbf{L}_{\text{sc}}^{[0]}$ and $\mathbf{W}^{[0]}$ from the input data [S1, unofficial]. Compute $\mathbf{Y}^{[0]} = y(\mathbf{L}_{\text{sc}}^{[0]})$ [S1, unofficial].

2. **Iterate for levels $k = 1$ to $K$:**
   
   a. **Compute Luminance Mask $\mathbf{Y}^{[k]}$:**
      Compute the luminance of the scattering image from the previous level:
      $$ \mathbf{Y}^{[k]} = y(\mathbf{L}_{\text{sc}}^{[k-1]}) $$
      [S1, unofficial]

   b. **Compute Spread-width Chain $\mathbf{W}^{[k]}$:**
      Perform a luminance-weighted average of the previous spread width $\mathbf{W}^{[k-1]}$ using the current luminance mask $\mathbf{Y}^{[k]}$:
      $$ \mathbf{W}^{[k]} = \frac{\left( \mathbf{Y}^{[k]} \cdot \mathbf{W}^{[k-1]} \right) * \mathcal{U}}{\mathbf{Y}^{[k]} * \mathcal{U}} $$
      [S1, unofficial]

   c. **Compute Masking Threshold $T$:**
      Calculate the threshold distance $T$ for the current level:
      $$ T = c \cdot 2^{k-1} $$
      [S1, unofficial]

   d. **Compute Auxiliary Mask $\mathbf{M}^{[k]}$:**
      Use the spread width from the *previous* level $\mathbf{W}^{[k-1]}$ to determine the mask:
      $$ \mathbf{M}^{[k]} = \text{smoothstep}\left(T, (1 + \varepsilon) \cdot T, \mathbf{W}^{[k-1]}\right) $$
      [S1, unofficial]

   e. **Compute Scattering Image $\mathbf{L}_{\text{sc}}^{[k]}$:**
      Apply the mask to the previous scattering image and convolve with the Gaussian filter:
      $$ \mathbf{L}_{\text{sc}}^{[k]} = \left( \mathbf{M}^{[k]} \cdot \mathbf{L}_{\text{sc}}^{[k-1]} \right) * \mathcal{G} $$
      [S1, unofficial]

**Parameters:**
- $c$: Constant from Equation 3.7 of [S1, unofficial]. Not given in the sources.
- $\varepsilon$: Width parameter for masking. Typical value $\varepsilon = 2$ reduces flickering [S1, unofficial]. Values $\varepsilon < 0.5$ cause high flickering [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Division by Zero:** In the luminance-weighted average for $\mathbf{W}^{[k]}$, if the denominator $\mathbf{Y}^{[k]} * \mathcal{U}$ is zero (no bright pixels in the neighborhood), the result is undefined [G]. A safe fallback is to copy $\mathbf{W}^{[k-1]}$ or set to 0 [G].
- **Flickering:** Low values of $\varepsilon$ (specifically $\varepsilon < 0.5$) cause visible flickering when moving around the scene [S1, unofficial].
- **Masking Logic:** The mask $\mathbf{M}^{[k]}$ depends on $\mathbf{W}^{[k-1]}$, not $\mathbf{W}^{[k]}$ [S1, unofficial]. This ensures the mask corresponds to the relation between spreading width and level $k$ in the neighborhood [S1, unofficial].

## Subtopics

### [Luminance Weighted MIP Map Filtering](9099-luminance-weighted-mip-map-filtering.md)
Luminance Weighted MIP Map Filtering is a technique to reduce illumination leaking artifacts in Gaussian MIP map blurring by using an auxiliary mask $\mathbb{M}$ to prevent low-standard-deviation pixels from propagating to high MIP levels. It differs from its parent [Spread-space MIP chain with luminance weighting](9093-spread-space-mip-chain-with-luminance-weighting.md) by masking out low-$W$ pixels before blurring, preventing illumination leaking seen in naive MIP maps where bright low-$W$ objects leak into dark high-$W$ backgrounds.

## Operating characteristics
- **Time Complexity:** $O(K \cdot N)$ where $N$ is the number of pixels, assuming fixed-size filters $\mathcal{G}$ and $\mathcal{U}$ [G]. Each level requires a few convolutions and element-wise operations [G].
- **Memory Complexity:** $O(K \cdot N)$ to store the MIP chains $\mathbf{L}_{\text{sc}}$, $\mathbf{W}$, $\mathbf{Y}$, and $\mathbf{M}$ [G].
- **Strengths:** Reduces illumination leaking artifacts by preventing low-standard-deviation pixels from propagating to high MIP levels [S1, unofficial]. Luminance weighting prioritizes brighter pixels for spreading width averaging [S1, unofficial].
- **Limitations:** Requires storing additional auxiliary chains ($\mathbf{W}, \mathbf{Y}, \mathbf{M}$) [G]. Sensitive to the parameter $\varepsilon$; incorrect values cause flickering [S1, unofficial].
- **When to use:** High-contrast scenes where naive Gaussian blurring causes illumination leaking [G].
- **When not to use:** Scenes with uniform lighting or low contrast, where the overhead may not be justified [G].

## Background
This method is based on the approach by Elek et al. [2013] as described in [S1, unofficial]. It was developed to address artifacts in real-time light transport in analytically integrable participating media, specifically when using MIP maps for filtering [S1, unofficial]. The technique combines spread-space MIP construction with luminance weighting to improve visual quality [S1, unofficial].

## Sources
[S1] PDF Real-Time Light Transport in Analytically Integrable Participating Media, https://cgg.mff.cuni.cz/~tomas/publications/pub-thesis-real-time-scattering/thesis.pdf, fetched 2026-10-29 (unofficial)
