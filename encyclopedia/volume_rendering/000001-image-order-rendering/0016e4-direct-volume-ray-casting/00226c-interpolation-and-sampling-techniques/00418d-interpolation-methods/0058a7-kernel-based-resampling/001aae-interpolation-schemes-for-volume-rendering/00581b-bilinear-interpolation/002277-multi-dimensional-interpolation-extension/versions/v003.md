# Multi-Dimensional Interpolation Extension

Also called: 3D Kernel Extension Methods, Separable and Radial Kernel Extension

## Overview
This method extends one-dimensional reconstruction kernels to higher dimensions for volume rendering [S1, unofficial]. It constructs a three-dimensional filter from a one-dimensional base kernel to correctly reconstruct signals on a regular mesh, distinguishing itself from radial extension methods by avoiding sample frequency ripple artifacts on structured grids [S1, unofficial].

## How it works
### Inputs and Outputs
*   **Input:** A one-dimensional reconstruction kernel function $h'(u)$, where $u$ represents the distance from the kernel center [S1, unofficial].
*   **Output:** A three-dimensional kernel function $h(x, y, z)$ evaluated at offset coordinates $(x, y, z)$ relative to the sample center [S1, unofficial].

### Data Structures
Not given in the sources.

### Steps and Formulas
**Method 1: Separable Extension (Recommended)**
This method treats each coordinate as an independent variable [S1, unofficial]. It is the correct way to extend filters to higher dimensions for regular meshes [S1, unofficial].
1. Evaluate the one-dimensional kernel for each coordinate offset: $h'(x)$, $h'(y)$, and $h'(z)$ [S1, unofficial].
2. Multiply the results to form the three-dimensional kernel:
   $$h(x, y, z) = h'(x) \times h'(y) \times h'(z)$$ [S1, unofficial].

**Method 2: Radial Extension**
This method uses the Euclidean radius from the kernel center as the free variable [S1, unofficial]. This produces rotationally-symmetric kernels [S1, unofficial].
1. Calculate the radius $r$ from the kernel center:
   $$r = \sqrt{x^2 + y^2 + z^2}$$ [S1, unofficial].
2. Evaluate the one-dimensional kernel at the radius:
   $$h(x, y, z) = h'(\sqrt{x^2 + y^2 + z^2})$$ [S1, unofficial].

### Parameters
Not given in the sources.

### Edge Cases and Pitfalls
*   **Sample Frequency Ripple:** Method 2 suffers from sample frequency ripple on regular meshes [S1, unofficial]. Even when input samples have equal amplitude, the output field will not be flat; it will differ from the constant amplitude at points between the equal-valued input samples [S1, unofficial].
*   **Reconstruction Accuracy:** Rotationally-symmetric kernels (except for the Gaussian, which is also separable) do not correctly reconstruct a regular mesh [S1, unofficial]. Therefore, Method 2 should generally not be used for standard volume rendering on structured grids [S1, unofficial].

## Subtopics

### Affine Transformation Interpolation
**Definition:** This variant applies separable kernels to enable two-pass and three-pass image warping [S1, unofficial]. It is much less sensitive to kernel extent sizes because the computational amount is $O(n)$ for a kernel extent of $n$, compared to $O(n^3)$ for ray-casting methods [S1, unofficial].
**Procedure:** When using three-pass approaches, the order of passes is critical [S1, unofficial]. All magnification steps must be performed before any minification steps [S1, unofficial]. If minification precedes magnification, multiple samples are collapsed into a single sample during minification and cannot be separated during subsequent passes, leading to signal loss [S1, unofficial].
**Usage:** Typically employs high-quality reconstruction kernels for all image generation to retain maximum signal information [S1, unofficial].

## Operating characteristics
*   **Complexity:** Ray-casting methods require $O(n^3)$ computation for a kernel extent of $n$, whereas affine transformation methods using separable kernels reduce this to $O(n)$ [S1, unofficial].
*   **Strengths:** Method 1 correctly reconstructs regular meshes and avoids the ripple artifacts inherent in Method 2 [S1, unofficial].
*   **Limitations:** Affine transformation methods are much less sensitive to kernel extent sizes than ray-casting methods [S1, unofficial]. However, affine methods require careful pass ordering to avoid information loss [S1, unofficial].
*   **When to use:** Use Method 1 for all regular mesh reconstructions [S1, unofficial]. Use Affine Transformation Interpolation when performing image warping or non-axis-aligned rendering where computational efficiency is prioritized and high-quality kernels are required [S1, unofficial].

## Background
The extension of one-dimensional kernels to higher dimensions is described as having two primary methods [S1, unofficial]. Volume renderers seldom generate images that are axis-aligned with identical sampling rates for input data and the image, so resampling and careful filter design are required [S1, unofficial]. Hanrahan [Hanrahan 90] is cited as a key contributor who considered these signal processing concerns, specifically regarding the ordering of magnification and minification in affine transformations [S1, unofficial].

## Sources
[S1] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27 (unofficial)
