# Spread-space MIP chain with luminance weighting

Also called: Luminance-weighted MIP chain, Masked MIP chain

## Overview
This method constructs a multi-resolution pyramid for scattering images by combining spatial blurring with a luminance-weighted auxiliary mask to prevent illumination leaking artifacts. It differs from its parent [Streaming MIP Implementation (Basic)](1803-streaming-mip-implementation-basic.md) by introducing a spreading width chain $\mathbf{W}$ and a luminance mask $\mathbf{Y}$ that modulate the Gaussian blur, ensuring that low-standard-deviation (sharp) pixels do not incorrectly propagate brightness to high MIP levels. It differs from [Conical MImP](7656-conical-mimp.md) by using a separable Gaussian blur with explicit masking rather than a conical volume approximation.

## How it works
**Inputs:**
- $\mathbf{L}_{\text{sc}}^{[0]}$: The input scattering image at MIP level 0.
- $\mathbf{W}^{[0]}$: The initial spreading width image (standard deviation map) at level 0.
- $K$: The maximum MIP level.
- $c$: A constant defining the base threshold distance (from Equation 3.7 of [S1, unofficial]).
- $\varepsilon$: The width parameter for the smoothstep masking function.

**Outputs:**
- $\mathbf{L}_{\text{sc}}^{[0...K]}$: The MIP chain of scattering images.
- $\mathbf{W}^{[0...K]}$: The MIP chain of spreading widths.
- $\mathbf{Y}^{[0...K]}$: The auxiliary luminance weighting mask chain.
- $\mathbf{M}^{[1...K]}$: The auxiliary masking chain (computed on-the-fly or stored).

**Data Structures:**
- Images are 2D arrays of floating-point values.
- $\mathbf{L}_{\text{sc}}$: RGB color values.
- $\mathbf{W}$: Single-channel scalar values (spreading width).
- $\mathbf{Y}$: Single-channel scalar values (luminance).
- $\mathbf{M}$: Single-channel scalar values (mask, 0.0 to 1.0).

**Definitions:**
- $\mathcal{G}$: Discrete Gaussian filter kernel.
- $\mathcal{U}$: Uniform distribution convolution kernel (averaging filter) of the same size as $\mathcal{G}$.
- $y(\mathbf{C})$: Function computing absolute RGB luminance from color $\mathbf{C}$. Standard formula: $y = 0.2126 R + 0.7152 G + 0.0722 B$.
- $\text{smoothstep}(e_1, e_2, x)$: Smooth interpolation function. Returns 0 if $x \le e_1$, 1 if $x \ge e_2$, and smooth interpolation between 0 and 1 if $e_1 < x < e_2$.

**Algorithm Steps:**

1. **Initialize Level 0:**
   Set $\mathbf{L}_{\text{sc}}^{[0]}$, $\mathbf{W}^{[0]}$, and compute $\mathbf{Y}^{[0]} = y(\mathbf{L}_{\text{sc}}^{[0]})$.

2. **Iterate for levels $k = 1$ to $K$:**
   
   a. **Compute Luminance Mask $\mathbf{Y}^{[k]}$:**
      Compute the luminance of the scattering image from the previous level:
      $$ \mathbf{Y}^{[k]} = y(\mathbf{L}_{\text{sc}}^{[k-1]}) $$
      [G] Ensure $\mathbf{Y}^{[k]}$ is clamped to non-negative values.

   b. **Compute Spread-width Chain $\mathbf{W}^{[k]}$:**
      Perform a luminance-weighted average of the previous spread width $\mathbf{W}^{[k-1]}$ using the current luminance mask $\mathbf{Y}^{[k]}$:
      $$ \mathbf{W}^{[k]} = \frac{\left( \mathbf{Y}^{[k]} \cdot \mathbf{W}^{[k-1]} \right) * \mathcal{U}}{\mathbf{Y}^{[k]} * \mathcal{U}} $$
      [G] Handle division by zero in the denominator where $\mathbf{Y}^{[k]} * \mathcal{U} \approx 0$ by setting $\mathbf{W}^{[k]}$ to 0 or the previous level's value.

   c. **Compute Masking Threshold $T$:**
      Calculate the threshold distance $T$ for the current level:
      $$ T = c \cdot 2^{k-1} $$

   d. **Compute Auxiliary Mask $\mathbf{M}^{[k]}$:**
      Use the spread width from the *previous* level $\mathbf{W}^{[k-1]}$ to determine the mask:
      $$ \mathbf{M}^{[k]} = \text{smoothstep}\left(T, (1 + \varepsilon) \cdot T, \mathbf{W}^{[k-1]}\right) $$
      [G] The smoothstep function can be implemented as:
      ```python
      def smoothstep(e1, e2, x):
          if x <= e1: return 0.0
          if x >= e2: return 1.0
          t = (x - e1) / (e2 - e1)
          return t * t * (3 - 2 * t)
      ```

   e. **Compute Scattering Image $\mathbf{L}_{\text{sc}}^{[k]}$:**
      Apply the mask to the previous scattering image and convolve with the Gaussian filter:
      $$ \mathbf{L}_{\text{sc}}^{[k]} = \left( \mathbf{M}^{[k]} \cdot \mathbf{L}_{\text{sc}}^{[k-1]} \right) * \mathcal{G} $$
      [G] The multiplication $\mathbf{M}^{[k]} \cdot \mathbf{L}_{\text{sc}}^{[k-1]}$ is element-wise. The convolution $* \mathcal{G}$ is a separable 2D Gaussian blur.

**Parameters:**
- $c$: Constant from Equation 3.7 of [S1, unofficial]. Not given in the sources.
- $\varepsilon$: Width parameter for masking. Typical value $\varepsilon = 2$ reduces flickering. Values $\varepsilon < 0.5$ cause high flickering.

**Edge Cases and Pitfalls:**
- **Division by Zero:** In the luminance-weighted average for $\mathbf{W}^{[k]}$, if the denominator $\mathbf{Y}^{[k]} * \mathcal{U}$ is zero (no bright pixels in the neighborhood), the result is undefined. [G] A safe fallback is to copy $\mathbf{W}^{[k-1]}$ or set to 0.
- **Flickering:** Low values of $\varepsilon$ (specifically $\varepsilon < 0.5$) cause visible flickering when moving around the scene. [S1, unofficial]
- **Masking Logic:** The mask $\mathbf{M}^{[k]}$ depends on $\mathbf{W}^{[k-1]}$, not $\mathbf{W}^{[k]}$. This ensures the mask corresponds to the relation between spreading width and level $k$ in the neighborhood.

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(K \cdot N)$ where $N$ is the number of pixels, assuming fixed-size filters $\mathcal{G}$ and $\mathcal{U}$. Each level requires a few convolutions and element-wise operations.
- **Memory Complexity:** $O(K \cdot N)$ to store the MIP chains $\mathbf{L}_{\text{sc}}$, $\mathbf{W}$, $\mathbf{Y}$, and $\mathbf{M}$.
- **Strengths:** Reduces illumination leaking artifacts by preventing low-standard-deviation pixels from propagating to high MIP levels. Luminance weighting prioritizes brighter pixels for spreading width averaging.
- **Limitations:** Requires storing additional auxiliary chains ($\mathbf{W}, \mathbf{Y}, \mathbf{M}$). Sensitive to the parameter $\varepsilon$; incorrect values cause flickering.
- **When to use:** High-contrast scenes where naive Gaussian blurring causes illumination leaking.
- **When not to use:** Scenes with uniform lighting or low contrast, where the overhead may not be justified.

## Background
This method is based on the approach by Elek et al. [2013] as described in [S1, unofficial]. It was developed to address artifacts in real-time light transport in analytically integrable participating media, specifically when using MIP maps for filtering. The technique combines spread-space MIP construction with luminance weighting to improve visual quality.

## Sources
[S1] PDF Real-Time Light Transport in Analytically Integrable Participating Media, https://cgg.mff.cuni.cz/~tomas/publications/pub-thesis-real-time-scattering/thesis.pdf, fetched 2026-10-29 (unofficial)
