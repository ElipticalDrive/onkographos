# Reparametrized Integral Gradients

Also called: Bias Removal via Reparameterization, Loubet et al. method, Moving visibility discontinuities, Path Space Sampling, Reparameterization for Discontinuous Integrands, Reparameterized Differentiable Path Tracing

## Overview
Reparametrized Integral Gradients is a technique for differentiable rendering that eliminates the bias in gradients caused by visibility discontinuities moving with scene parameters [S1][S2, unofficial][S3]. Unlike edge sampling methods that explicitly detect and sample discontinuity boundaries, this method applies a change of variables to transform the integration domain such that discontinuities become static (parameter-independent) [S1][S4, unofficial]. This allows the gradient to be computed by differentiating under the integral sign, effectively removing the boundary term required by the Reynolds Transport Theorem [S2, unofficial][S3]. It is distinguished from [Reparameterized Volume Sampling](3309-reparameterized-volume-sampling.md) by its focus on spherical and boundary integrals via warping, and from [Boundary Path Integral Formulation](3389-boundary-path-integral-formulation.md) by its avoidance of explicit boundary term estimation [S2, unofficial].

## How it works
The method addresses the rendering integral $I(\pi)$ where $\pi$ represents scene parameters (e.g., object position, shape) that affect visibility. Direct differentiation yields a boundary term corresponding to the motion of the discontinuity $T(\pi)$ [S2, unofficial][S3]. Reparameterization introduces a transformation $T_\pi: \Omega \to \Omega$ (or $R$ for spherical domains) that maps a fixed domain to a warped domain, aligning the moving discontinuity with a fixed point in the transformed space [S1][S5, unofficial].

### Inputs and Outputs
- **Inputs**: Scene parameters $\pi$; primal integrand $f(\omega)$ defined over domain $\Omega$ (e.g., unit sphere $S^2$ for pixel integrals); transformation function $R(\omega, \pi)$ or $T(\omega, \pi)$.
- **Outputs**: Gradient $\frac{\partial I}{\partial \pi}$; primal value $I$ (optional, if computed simultaneously).

### Data Structures
- **Transformation Field**: A function or lookup table defining the warp $R(\omega, \pi)$. For spherical rotations, this is defined by an axis and angle derived from the discontinuity motion [S5, unofficial].
- **Sample Points**: Uniform or importance-sampled points $\omega$ in the fixed domain $\Omega$ [S1].

### Steps
1. **Identify Discontinuities**: Determine the location of visibility discontinuities $T(\pi)$ in the original domain $\Omega$ as a function of parameters $\pi$ [S2, unofficial].
2. **Construct Transformation**: Define a differentiable transformation $R(\omega, \pi)$ such that the discontinuity point in the transformed domain, $\omega_0$, is independent of $\pi$. This requires satisfying the condition:
   $$
   \frac{\partial R}{\partial \pi} \approx \frac{\partial T}{\partial \pi}
   $$
   at the discontinuity [S5, unofficial]. For spherical integrals, $R$ is a rotation that maps a fixed direction $\omega_0$ to the moving discontinuity $T(\pi)$ [S5, unofficial].
3. **Rewrite Integral**: Transform the integral from $\Omega$ to the fixed domain $\Omega$:
   $$
   I(\pi) = \int_{\Omega} f(R(\omega, \pi)) \left| \det \frac{\partial R}{\partial \omega} \right| d\omega
   $$
   For volume-preserving transformations like spherical rotations, $\left| \det \frac{\partial R}{\partial \omega} \right| = 1$ [S5, unofficial].
4. **Differentiate**: Compute the gradient by differentiating under the integral sign:
   $$
   \frac{\partial I}{\partial \pi} = \int_{\Omega} \frac{\partial}{\partial \pi} \left[ f(R(\omega, \pi)) \left| \det \frac{\partial R}{\partial \omega} \right| \right] d\omega
   $$
   Since the domain $\Omega$ and the discontinuity location $\omega_0$ are now static, no boundary term is needed [S2, S5, unofficial].
5. **Estimate**: Use Monte Carlo sampling over $\Omega$ to estimate the integral and its gradient [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reparameterized_gradient(f, pi, samples, transform):
    """
    Estimate gradient of integral I(pi) = int f(omega) domega
    using reparameterization to make discontinuities static.
    
    Args:
        f: Integrand function f(omega)
        pi: Scene parameters
        samples: Array of sample points omega in fixed domain Omega
        transform: Function R(omega, pi) returning warped omega'
        
    Returns:
        gradient: Estimated dI/dpi
    """
    grad_sum = 0.0
    n_samples = len(samples)
    
    for omega in samples:
        # 1. Warp sample to original domain
        omega_prime = transform(omega, pi)
        
        # 2. Evaluate integrand at warped point
        f_val = f(omega_prime)
        
        # 3. Compute Jacobian determinant |det(dR/domega)|
        # For spherical rotations, this is 1. For general warps, compute numerically or analytically.
        jacobian = compute_jacobian_determinant(transform, omega, pi)
        
        # 4. Compute derivative of integrand with respect to pi
        # d/dpi [ f(R(omega, pi)) * |det| ]
        # Using chain rule: df/domega_prime * domega_prime/dpi
        df_domega = compute_gradient_of_f(omega_prime)
        domega_prime_dpi = compute_derivative_of_transform(transform, omega, pi)
        
        # Gradient contribution
        # Note: If jacobian depends on pi, add term: f * d|det|/dpi
        d_jacobian_dpi = compute_derivative_of_jacobian(transform, omega, pi)
        
        integrand_grad = (df_domega @ domega_prime_dpi) * jacobian + f_val * d_jacobian_dpi
        
        grad_sum += integrand_grad
        
    # Average over samples
    gradient = grad_sum / n_samples
    return gradient
```

### Parameters and Typical Values
- **Transformation Accuracy**: The approximation $\frac{\partial R}{\partial \pi} \approx \frac{\partial T}{\partial \pi}$ is critical. In practice, $R$ is often constructed by tracing auxiliary rays to estimate the discontinuity motion [S5, unofficial].
- **Sampling**: Uniform sampling on the sphere $S^2$ is common for pixel integrals [S5, unofficial].

### Edge Cases and Pitfalls
- **Specular Materials**: The method does not support perfectly specular materials because the discontinuity behavior is complex and non-differentiable in the same way [S2, unofficial].
- **Dirac Delta Sources**: Degenerate light sources with Dirac delta functions are not supported [S2, unofficial].
- **Multiple Discontinuities**: If multiple discontinuities exist within the support of the integrand, a single transformation may not align all of them to fixed points, leading to residual bias [S2, unofficial].
- **Approximation Error**: The gradient is an approximation if the transformation does not perfectly match the discontinuity motion. Gradients may not always be accurate [S2, unofficial].

## Subtopics

### Spherical Rotation Reparameterization
Spherical Rotation Reparameterization is a specific instance of Reparametrized Integral Gradients applied to pixel integrals over the unit sphere $S^2$ [S5, unofficial]. It uses spherical rotations $R(\omega, \pi)$ to warp the domain, making the discontinuity point $\omega_0$ independent of $\pi$ [S5, unofficial]. It is chosen when differentiating pixel contributions with respect to object positions or shapes where the dominant discontinuities are occlusion edges [S5, unofficial]. It builds on [Reparametrized Integral Gradients](6785-reparametrized-integral-gradients.md) but is limited to spherical domains and rotation-based warps [S5, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N)$ per sample, where $N$ is the number of samples. Requires evaluating the transformation and its derivative for each sample [S1][S5, unofficial].
- **Memory Complexity**: $O(1)$ auxiliary memory per sample, assuming the transformation is computed on the fly or stored efficiently [S5, unofficial].
- **Performance**: Computationally efficient compared to edge sampling, which requires finding and sampling edges explicitly [S2, unofficial]. However, variance can be high if the transformation does not perfectly align discontinuities [S1].
- **Strengths**: Eliminates bias from moving discontinuities without explicit edge detection; compatible with automatic differentiation [S1, S3].
- **Limitations**: Approximate gradients; does not handle specularities or multiple discontinuities well; requires careful construction of the warp field [S2, S5, unofficial].
- **When to Use**: When differentiating with respect to parameters that cause visibility changes (e.g., object translation) and explicit edge sampling is too costly or complex [S2, unofficial][S3].
- **When Not to Use**: For scenes with perfect specular reflections or complex light sources with Dirac deltas [S2, unofficial].

## Background
Reparametrized Integral Gradients was introduced by Loubet et al. in "Reparameterizing Discontinuous Integrands for Differentiable Rendering" at SIGGRAPH Asia 2019 [S4, S5, unofficial]. It was developed to address the bias in gradients caused by visibility discontinuities in differentiable rendering, which naive differentiation fails to capture correctly [S3]. The method is part of a broader family of techniques including edge sampling and path space sampling [S3]. It is implemented in differentiable rendering systems like Mitsuba 3 [S4, unofficial].

## Sources
[S1] Physics-based differentiable rendering, https://doi.org/10.1145/3388769.3407454, fetched 2026-10-29
[S2] Differentiable Rendering: A Survey, https://doi.org/10.48550/arxiv.2006.12057, fetched 2026-10-26 (unofficial)
[S3] DR.JIT, https://doi.org/10.1145/3528223.3530099, fetched 2026-10-26
[S4] Differentiable ray tracing — from paths to gradients | Bai Liping, https://bailiping.com/differentiable-ray-tracing/, fetched 2026-10-28 (unofficial)
[S5] A Survey on Physics-based Differentiable Rendering - arXiv.org, https://arxiv.org/pdf/2504.01402, fetched 2026-10-29 (unofficial)
