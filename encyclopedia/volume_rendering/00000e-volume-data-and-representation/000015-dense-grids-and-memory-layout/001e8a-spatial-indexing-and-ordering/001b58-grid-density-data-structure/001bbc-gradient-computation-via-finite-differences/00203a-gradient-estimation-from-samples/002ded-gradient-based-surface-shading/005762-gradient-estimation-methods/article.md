# Gradient Estimation Methods

Also called: Central difference, Cubic-spline gradient, Intermediate difference

## Overview
Gradient estimation methods approximate the volume gradient at sample locations to provide surface normals for shading iso-surfaces, solving the problem of how to derive accurate normals from discrete scalar field data [S1, unofficial]. They differ from their siblings by focusing on the accuracy of the gradient vector itself rather than just the surface location, with the choice of method significantly impacting visual artifacts like ripple effects when sample distances are large [S1, unofficial]. This family builds on [Gradient-Based Surface Shading](5648-gradient-based-surface-shading.md).

## Shared principles
The family shares the goal of estimating the gradient $\nabla f$ of a scalar field $f$ to approximate the surface normal $\mathbf{n} = \frac{\nabla f}{\|\nabla f\|}$ [G]. The "desired" gradient is defined as the derivative of the step response at the actual edge location, which equals the maximum value of the point-spread function occurring at $x=0$ [S1, unofficial].

For the finite difference methods (central and intermediate), the process involves two steps:
1. Calculate the gradient at specific sample locations [S1, unofficial].
2. For arbitrary locations, linearly interpolate the calculated gradient at or between samples [S1, unofficial].

The cubic-spline method computes the derivative of the cubic-spline interpolation function directly [S1, unofficial].

A critical shared pitfall is that the estimated gradient depends heavily on the sample location and distance [S1, unofficial]. This dependency causes severe ripple artifacts in visualization when the sample distance is large [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def estimate_gradient(samples, sample_positions, query_position, method):
    # Step 1: Calculate gradient at sample locations
    # Not given in the sources.
    gradients_at_samples = compute_gradients_at_samples(samples, sample_positions)
    
    # Step 2: Evaluate at arbitrary location
    if method in ("central", "intermediate"):
        # Linearly interpolate the calculated gradient at or between samples
        # Not given in the sources.
        return linear_interpolate(gradients_at_samples, sample_positions, query_position)
    elif method == "cubic_spline":
        # Compute the derivative of the cubic-spline interpolation function
        # Not given in the sources.
        return compute_cubic_spline_derivative(samples, sample_positions, query_position)
```

## Subtopics

### Central Difference
The central difference method calculates the gradient at sample locations and linearly interpolates for arbitrary positions [S1, unofficial]. It yields a worse approximation of the derivative when the edge lies exactly between two samples compared to the intermediate difference method [S1, unofficial]. At sampling locations, it produces the same results as the intermediate difference method, but the estimated gradient at the edge location is not the correct derivative of the continuous step response [S1, unofficial].

### Intermediate Difference
The intermediate difference method is more accurate than central difference when the edge lies exactly between two samples (e.g., sample location = 0.5) [S1, unofficial]. It is generally less sensitive to noise in the data compared to linear and cubic-spline functions [S1, unofficial]. When the estimated edge location is identical to the exact edge location, the intermediate difference method provides a better approximation of the derivative of the step response at other sample locations [S1, unofficial].

### Cubic-Spline Gradient
This method computes the derivative of the cubic-spline interpolation function [S1, unofficial]. It is generally very inaccurate as an approximation of the derivative at sample points [S1, unofficial]. However, it improves between samples due to overshoot, where the estimated gradient can become larger than the desired derivative [S1, unofficial].

### Linear Approximation Gradient
A gradient approximation method for differentiable surface splatting that is invariant to the hyper-parameter $\sigma_k$ (RBF standard deviation), unlike RBF-based gradients which are highly sensitive and prone to local minima [S2]. RBF-based gradients suffer from diminishing gradients for distant pixels when $\sigma_k$ is small, causing convergence issues [S2]. When $\sigma_k$ is large, the gradient norm can increase outside the ellipse boundary, causing optimization to fall into local minima in multi-point scenarios [S2]. Unlike RBF-based gradients which evaluate 1D gradients separately (considering only pixels in the same row/column), this method evaluates the gradient for all dimensions of $\mathbf{p}$ jointly [S2]. It also considers a set of occluded and occluding ellipses projected to pixel $\mathbf{x}$, leading to more accurate gradients and encouraging noisy points to move onto the surface [S2].

## Background
Originated in the context of iso-surface volume rendering for medical applications, these methods were evaluated for speed and accuracy [S1, unofficial]. The analysis highlights that while fixed volume gradients make the choice of re-sampling function less influential on visual appearance, the gradient estimation method itself is crucial for avoiding artifacts [S1, unofficial].

## Sources
[S1] Iso-Surface Volume Rendering : speed and accuracy for medical applications, https://doi.org/10.3990/1.9789036513982, fetched 2026-10-26 (unofficial)
[S2] Differentiable surface splatting for point-based geometry processing, https://doi.org/10.1145/3355089.3356513, fetched 2026-10-26
