# Gaussian Interpolation

Also called: Gaussian Filter

## Overview
Gaussian interpolation is a volumetric reconstruction method that uses a Gaussian function as an interpolation kernel to estimate field values at arbitrary points from discrete samples [S1, unofficial][S2, unofficial]. It solves the problem of reconstructing a continuous volume from a grid or point set while providing smooth, band-limited results [S1, unofficial]. It is distinguished from its parent, the [Normalized 3D Gaussian Function](1064-normalized-3d-gaussian-function.md), by its specific role as a separable, rotationally symmetric interpolation filter rather than a primitive shape definition [S2, unofficial]. Unlike the [2D Gaussian Probability Density (Standard)](3466-2d-gaussian-probensity-standard.md), it is explicitly designed for multi-dimensional spatial reconstruction in volume rendering contexts [S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A set of discrete sample values $V$ at grid points or positions, and a query position $\mathbf{x} = (x, y, z)$ [G].
- Output: The interpolated scalar value $f(\mathbf{x})$ [G].

**Data Structures:**
- The kernel is defined by a width parameter $\sigma$ (or $a$) and a scale factor $b$ [S1, unofficial][S2, unofficial].
- In 3D, the kernel is often implemented via separability, storing 1D kernel weights or computing them on the fly [S2, unofficial].

**Formulas:**
The 1D Gaussian kernel is defined as [S2, unofficial]:
$$f_{\text{Gaussian}}(z) = e^{-\frac{z^2}{2\sigma^2}}$$
where $\sigma$ controls the width of the filter [S2, unofficial].

In the context of grid-based interpolation described in [S1, unofficial][S3, book], the kernel is parameterized by $a$ (width) and $b$ (scale factor). The 3D Gaussian kernel is given by [S3, book]:
$$h(u, v, w) = b \cdot e^{-a(u^2 + v^2 + w^2)}$$
where $u, v, w$ are the distances of the sample point from the lower, left, rear voxel in the containing cell [S1, unofficial]. Note: [S1, unofficial] also erroneously cites a linear interpolation formula $f(x,y,z) = (1-w)f(u,v,0) + wf(u,v,1)$ as Gaussian in the same paragraph, but explicitly identifies the exponential form above as the Gaussian [S1, unofficial]. When sources disagree on the exponent coefficient, [S1, unofficial] uses $a^2$ while [S3, book] uses $a$ [S1, unofficial][S3, book].

**Parameters:**
- $\sigma$ (or $a$): Determines the width of the filter. Larger values result in more smoothing [S1, unofficial][S2, unofficial].
- $b$: A scale factor [S1, unofficial].

**Steps:**
1. For a query point $\mathbf{x}$, identify the surrounding grid cells or sample points [G].
2. Compute the distance $d$ from $\mathbf{x}$ to each sample point [G].
3. Evaluate the Gaussian kernel weight $w = e^{-\frac{d^2}{2\sigma^2}}$ for each sample [S2, unofficial].
4. Sum the weighted sample values and normalize if necessary (depending on whether the kernel is normalized to integrate to one or used as a raw filter) [S1, unofficial][S2, unofficial].

**Edge Cases and Pitfalls:**
- **Ripple Artifact:** The Gaussian introduces a slight ripple (about 0.1%) into an interpolated uniform function [S1, unofficial].
- **Blurring:** The filter distorts the passband by attenuating high frequencies, causing blurring in the output [S2, unofficial].
- **Infinite Support:** The Gaussian has infinite extent in spatial and frequency domains. It must be truncated for practical implementation, though it falls to zero quickly, minimizing ringing [S2, unofficial].
- **Uniform Signal Interpolation:** A Gaussian filter does not yield $f(x,y,z) = V(x,y,z)$ at grid points and does not interpolate a uniform $f(x,y,z)$ everywhere for uniform grid signals [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def gaussian_kernel_1d(dist, sigma):
    """Evaluates the 1D Gaussian kernel e^(-dist^2 / (2*sigma^2))."""
    return math.exp(-(dist ** 2) / (2 * sigma ** 2))

def gaussian_interpolate_1d(samples, query_pos, sigma, truncation_radius=None):
    """
    Interpolate a value at query_pos using a 1D Gaussian kernel.
    samples: list of (position, value) tuples.
    query_pos: float, the position to interpolate at.
    sigma: float, the width parameter.
    truncation_radius: float, optional cutoff distance for infinite support.
    """
    total_weight = 0.0
    total_value = 0.0
    
    for pos, val in samples:
        dist = query_pos - pos
        if truncation_radius is not None and abs(dist) > truncation_radius:
            continue
        weight = gaussian_kernel_1d(dist, sigma)
        total_weight += weight
        total_value += val * weight
        
    if total_weight == 0:
        return 0.0
    return total_value / total_weight

def gaussian_interpolate_3d_separable(grid, query_pos, sigma, truncation_radius=None):
    """
    Interpolate using separable 1D Gaussians along x, y, z axes.
    grid: 3D array of values.
    query_pos: tuple (x, y, z) of floats.
    sigma: float, the width parameter.
    truncation_radius: float, optional cutoff distance for infinite support.
    """
    total_weight = 0.0
    total_value = 0.0
    
    for i in range(len(grid)):
        for j in range(len(grid[0])):
            for k in range(len(grid[0][0])):
                dx = i - query_pos[0]
                dy = j - query_pos[1]
                dz = k - query_pos[2]
                dist_sq = dx**2 + dy**2 + dz**2
                if truncation_radius is not None and dist_sq > truncation_radius**2:
                    continue
                weight = gaussian_kernel_1d(dx, sigma) * gaussian_kernel_1d(dy, sigma) * gaussian_kernel_1d(dz, sigma)
                total_weight += weight
                total_value += grid[i][j][k] * weight
                    
    if total_weight == 0:
        return 0.0
    return total_value / total_weight
```

## Subtopics

### [Gaussian Reconstruction Filter](2852-gaussian-reconstruction-filter.md)
The Gaussian Reconstruction Filter is a spatial weighting function applied to jittered samples to improve edge quality and stability in volumetric rendering, specifically replacing box filters in froxel-based temporal anti-aliasing. It is also used in A-buffer methods to compute data weights based on distance from a node. It sets itself apart from its sibling, the [Gaussian Kernel Volumetric Primitive](4010-gaussian-kernel-volumetric-primitive.md), by acting as a post-sampling reconstruction filter for discrete jittered points rather than defining the primitive's shape itself.

### [Gaussian Kernel Volumetric Primitive](4010-gaussian-kernel-volumetric-primitive.md)
Gaussian Kernel Volumetric Primitive represents a 3D scene as a collection of volumetric primitives, where each primitive is a 3D Gaussian distribution parameterized by mean, covariance, and color. Unlike standard splatting methods that project Gaussians to 2D ellipses for rasterization, this method treats the Gaussian as a continuous density field in 3D space, enabling physically-based volume rendering (e.g., scattering, emission) via ray tracing. It solves the problem of efficiently simulating 3D structure details and volumetric effects without requiring explicit meshing or expensive neural network evaluations. It is distinguished from its parent [Gaussian Interpolation](2347-gaussian-interpolation.md) by its explicit use of the 3D Gaussian PDF as a volumetric density function rather than just an interpolation kernel, and from [Gaussian Reconstruction Filter](2852-gaussian-reconstruction-filter.md) by its focus on forward volumetric rendering and ray-marching integration.

### Gaussian Perturbation Kernel
A variant used in differentiable path tracing to handle shadow boundaries and reduce plateau artifacts [S4]. To compute the CDF, it integrates the PDF, treating each halfspace separately and normalizing the PDF on the positive halfspace to integrate to 0.5 [S4]. The kernel derivative with respect to parameters is given by $\frac{\partial \kernel}{\partial \parameters}(\offset) = \frac{-\offset}{\bandwidth^3\sqrt{2\pi}}\exp\left(\frac{-\offset^2}{2\bandwidth^2}\right)$, which is a translated version of the main text's Eq. 11, simplified by treating offset relative to parameters [S4].

## Operating characteristics
**Time and Memory Complexity:**
- The Gaussian kernel has infinite support, requiring truncation for practical implementation. The number of samples contributing to a query depends on the truncation radius and $\sigma$ [S2, unofficial].
- Separability allows 3D reconstruction to be performed as three 1D passes, reducing computational complexity compared to full 3D convolution [S2, unofficial].

**Measured Performance:**
- Not given in the sources.

**Scaling:**
- Performance scales with the number of samples within the truncation radius [S2, unofficial].

**Strengths and Limitations:**
- **Strengths:** It is the only filter that is both separable and rotationally symmetric [S2, unofficial]. It provides smooth interpolation with infinite continuity in the derivative [S1, unofficial]. It is widely used in point-tossing approaches like splatting because it allows integrating one dimension out, performing 2D reconstruction instead of 3D [S2, unofficial].
- **Limitations:** It introduces a slight ripple (0.1%) in uniform functions [S1, unofficial]. It attenuates high frequencies, causing blurring [S2, unofficial]. It does not preserve uniform values exactly [S1, unofficial]. Interpolation cannot restore sharp edges that existed in the original function prior to sampling [S1, unofficial].

**When to Use:**
- When a radially symmetric interpolation kernel is needed [S1, unofficial].
- For grids that assume the frequency spectrum is radially bandlimited [S1, unofficial].
- In splatting methods where separability and rotational symmetry are beneficial [S2, unofficial].

**When Not to Use:**
- When sharp edge preservation is critical, as interpolation inherently smooths the signal and requires non-linear filtering to restore edges [S1, unofficial].
- When exact interpolation of uniform data is required [S1, unofficial].

## Background
Gaussian interpolation is cited in the context of volume rendering overviews [S1, unofficial] and parallel volume rendering algorithms [S2, unofficial]. It is also used in modern differentiable rendering techniques, such as plateau-reduced differentiable path tracing [S4]. The method is often compared with box, linear, cubic, and cardinal spline filters [S1, unofficial].

## Sources
[S1] Overview of Volume Rendering, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S2] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27 (unofficial)
[S3] PDF Volume Visualization and Volume Graphics, https://www3.cs.stonybrook.edu/~mueller/teaching/cse332/volvisChapter.pdf, fetched 2026-10-28 (book)
[S4] Plateau-Reduced Differentiable Path Tracing, https://doi.org/10.1109/cvpr52729.2023.00417, fetched 2026-10-26
