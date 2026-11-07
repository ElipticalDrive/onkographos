# Gaussian Interpolation

Also called: Gaussian Filter, Gaussian Kernel Interpolation

## Overview
Gaussian Interpolation uses a Gaussian function as an interpolation kernel to reconstruct continuous signals from discrete samples [S1, unofficial]. It is distinguished from other filters by being the only one that is both separable and rotationally symmetric, a property that allows it to integrate one dimension out of the kernel to perform 2D reconstruction instead of 3D, making it suitable for point-tossing approaches like splatting [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Sample positions and values; a width parameter $\sigma$.
*   **Output:** Interpolated value at a query point.

**Data Structures:**
Not given in the sources.

**Steps:**
1.  Calculate the distance $z$ between the query point and the sample point.
2.  Compute the weight using the Gaussian function:
    $$f_{\text{Gaussian}}(z) = e^{-\frac{z^2}{2\sigma^2}}$$
    where $\sigma$ controls the width of the Gaussian [S1, unofficial].
3.  Aggregate the weighted sample values to produce the interpolated result.

**Parameters:**
*   $\sigma$: Controls the width of the Gaussian [S1, unofficial].

**Pitfalls:**
*   The kernel has infinite extent in both spatial and frequency domains [S1, unofficial].
*   It asymptotically falls toward zero quickly, allowing truncation without significant ringing, but this truncation requires careful handling of the near-zero area [S1, unofficial].
*   The filter distorts the passband by attenuating high-frequency components of the main replica, causing blurring in the output [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gaussian_interpolation(samples, query_point, sigma):
    """
    Interpolate value at query_point using Gaussian kernel.
    samples: list of (position, value) tuples.
    sigma: width parameter.
    """
    total_weight = 0.0
    total_value = 0.0
    
    for pos, val in samples:
        # Calculate distance z
        z = distance(pos, query_point)
        
        # Compute Gaussian weight
        # f(z) = exp(-z^2 / (2 * sigma^2))
        weight = math.exp(-(z ** 2) / (2 * sigma ** 2))
        
        total_weight += weight
        total_value += val * weight
        
    if total_weight == 0:
        return 0.0
        
    return total_value / total_weight
```

## Subtopics

### [Gaussian Kernel Volumetric Primitive](4010-gaussian-kernel-volumetric-primitive.md)
A method for volumetric rendering that represents a scene as a set of 3D Gaussian distributions, parameterized by mean (position), covariance (shape/direction), and color, allowing for efficient simulation of 3D structure details without meshing. It differs from traditional mesh-based reconstruction by avoiding meshing processes and from NeRF by offering faster training and rendering speeds while maintaining comparable quality. Unlike other kernel methods like Epanechnikov, Gaussian kernels have infinite support, which requires clipping to maintain efficiency.

### Gaussian Reconstruction Filter
A weighting filter applied to jittered sub-froxel samples to improve edge quality and stability compared to a box filter, based on distance from the froxel center [S2, unofficial]. The weight $W(x)$ for a sample at distance $x$ from the center of the froxel is calculated as:
$$W(x) = e^{-2.29x^2}$$
[S2, unofficial].

### Gaussian Perturbation Kernel
Used in differentiable path tracing to compute the Cumulative Distribution Function (CDF) by integrating the Probability Density Function (PDF), treating each halfspace separately and normalizing the PDF on the positive halfspace to integrate to 0.5 [S3]. The derivative of the kernel with respect to parameters is given by:
$$\frac{\partial \kernel}{\partial \parameters}(\offset) = \frac{-\offset}{\bandwidth^3\sqrt{2\pi}}\exp\left(\frac{-\offset^2}{2\bandwidth^2}\right)$$
[S3].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Separability and rotational symmetry allow for computational savings by reducing 3D reconstruction to 2D reconstruction in splatting methods [S1, unofficial]. Truncation is possible without significant ringing due to rapid asymptotic decay [S1, unofficial].
*   **Limitations:** Infinite extent requires clipping or truncation. Attenuation of high frequencies causes blurring [S1, unofficial].
*   **When to use:** Point-tossing approaches like splatting where separability and rotational symmetry are beneficial [S1, unofficial].
*   **When not to use:** When high-frequency preservation is critical due to blurring artifacts [S1, unofficial].

## Background
Gaussian interpolation has been used in research for volume rendering, particularly in parallel, feed-forward algorithms [S1, unofficial]. It is applied in point-tossing approaches such as splatting [Westover 90] cited in [S1, unofficial]. In temporal anti-aliasing for volumetric rendering, a Gaussian filter fit to Blackman-Harris 3.3 is used for jittered samples [S2, unofficial]. In differentiable path tracing, Gaussian kernels are used for plateau reduction [S3].

## Sources
[S1] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27 (unofficial)
[S2] Temporal Anti-Aliasing for Real-Time Froxel-Based Volumetric Rendering - Advances in volumetric temporal anti-aliasing for real-time ren dering of participating media, https://openalex.org/W7218067689, fetched 2026-11-01 (unofficial)
[S3] Plateau-Reduced Differentiable Path Tracing, https://doi.org/10.1109/cvpr52729.2023.00417, fetched 2026-10-26
