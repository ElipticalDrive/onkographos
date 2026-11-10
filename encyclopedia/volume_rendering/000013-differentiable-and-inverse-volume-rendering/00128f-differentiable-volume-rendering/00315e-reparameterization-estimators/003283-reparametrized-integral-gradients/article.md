# Reparametrized Integral Gradients

Also called: Bias Removal via Reparameterization, Loubet et al. method, Moving visibility discontinuities, Path Space Sampling, Reparameterization for Discontinuous Integrands, Reparameterized Differentiable Path Tracing

## Overview
Reparametrized Integral Gradients is a technique for differentiable rendering that eliminates bias in gradients caused by moving visibility discontinuities by applying changes of variables to render the discontinuities static in the transformed domain [S1][S2, unofficial][S3]. Unlike explicit edge sampling methods that require detecting and sampling geometric boundaries, this method reparameterizes the integration domain (such as spherical domains for light transport) so that the discontinuity points become independent of the scene parameters $\pi$ [S2, S5, unofficial]. This allows the gradient to be computed via standard Monte Carlo integration over the interior, avoiding the boundary terms that arise when differentiating under the integral sign [S1][S5, unofficial]. It is distinct from [Reparameterized Volume Sampling](3309-reparameterized-volume-sampling.md) which focuses on density fields, and [Boundary Path Integral Formulation](3389-boundary-path-integral-formulation.md) which explicitly separates and samples boundary terms [S2, unofficial][S3].

## How it works
The method addresses the problem where the rendering integral $I(\pi)$ contains a discontinuity at a point $T(\pi)$ that depends on a parameter $\pi$ (e.g., object position or shape). Direct differentiation yields a boundary term that is difficult to estimate [S3]. The method transforms the integral into a domain where the discontinuity is stationary.

**Inputs:**
- Scene parameters $\pi$ (e.g., translation, rotation, scale).
- Primal integrand $f(\omega, \pi)$ defined over a domain $\Omega$ (e.g., the unit sphere $S^2$ for incoming light directions).
- Discontinuity location $T(\pi) \in \Omega$ which moves with $\pi$.

**Outputs:**
- Gradient estimate $\nabla_\pi I(\pi)$.

**Data Structures:**
- **Warp Field / Rotation Map $R(\omega, \pi)$**: A differentiable mapping from the parameter-independent domain $\Omega$ to the original domain. For spherical integrals, this is a rotation matrix or quaternion representation.
- **Sample $u \sim p(u)$**: Uniform samples on the parameter-independent domain (e.g., uniform directions on $S^2$).

**Steps:**
1. **Identify Discontinuity Motion**: Determine the location $T(\pi)$ of the visibility discontinuity (e.g., the direction of a silhouette edge or specular reflection) as a function of $\pi$ [S2, S5, unofficial].
2. **Construct Reparametrization Map**: Define a transformation $R(\omega, \pi)$ such that the discontinuity in the transformed domain is stationary. Specifically, find $R$ such that there exists a fixed point $\omega_0$ where $R(\omega_0, \pi) = T(\pi)$ for all $\pi$ [S5, unofficial].
3. **Approximate Motion**: Choose $R$ to approximate the motion of the discontinuity. The condition is that the derivative of the rotation with respect to $\pi$ matches the derivative of the discontinuity path:
   $$
   \frac{\partial R}{\partial \pi} \approx \frac{\partial T}{\partial \pi}
   $$
   This ensures that the transformed discontinuity point $\omega_0$ is independent of $\pi$ [S5, unofficial].
4. **Transform Integral**: Rewrite the integral $I$ using the change of variables $\omega' = R(\omega, \pi)$:
   $$
   I(\pi) = \int_{\Omega} f(R(\omega, \pi), \pi) p(\omega) d\omega
   $$
   where $p(\omega)$ is the uniform measure on $\Omega$ [S5, unofficial].
5. **Differentiate**: Since the discontinuity $\omega_0$ is now static (independent of $\pi$), the derivative can be moved inside the integral without generating a boundary term:
   $$
   \nabla_\pi I(\pi) = \int_{\Omega} \nabla_\pi \left[ f(R(\omega, \pi), \pi) \right] p(\omega) d\omega
   $$
   [S5, unofficial].
6. **Estimate**: Compute the Monte Carlo estimate by sampling $\omega$ uniformly from $\Omega$ and evaluating the gradient of the transformed integrand [S1][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reparameterized_gradient_integral(pi, num_samples):
    """
    Estimates the gradient of a rendering integral with respect to parameter pi
    using spherical rotation reparameterization.
    
    Args:
        pi: Scene parameter (scalar or vector).
        num_samples: Number of Monte Carlo samples.
        
    Returns:
        Gradient estimate.
    """
    gradient_sum = 0.0
    area = 4 * pi  # Area of unit sphere S^2
    
    for _ in range(num_samples):
        # 1. Sample uniform direction on parameter-independent domain
        omega = sample_uniform_sphere()
        
        # 2. Compute the discontinuity location T(pi) in original domain
        # This requires tracing a ray or computing geometry intersection
        T_pi = compute_discontinuity_direction(pi)
        
        # 3. Construct rotation R such that R(omega_0, pi) = T(pi)
        # We need to find omega_0 such that the rotation maps it to T(pi).
        # Alternatively, define R(omega, pi) that rotates omega towards T(pi)
        # based on the relative orientation.
        # A simple rotation aligning a reference axis to T(pi) can be used.
        # Let's assume we have a function that returns the rotated direction.
        omega_prime = rotate_direction(omega, pi, T_pi)
        
        # 4. Evaluate integrand at transformed point
        # f(omega_prime, pi) is the original integrand value
        f_val = evaluate_integrand(omega_prime, pi)
        
        # 5. Compute derivative of integrand with respect to pi
        # This includes the explicit dependence of f on pi and the implicit
        # dependence through omega_prime = R(omega, pi).
        # d/dpi f(R(omega, pi), pi) = df/dpi + df/domega_prime * d(omega_prime)/dpi
        
        # Explicit derivative of f with respect to pi (e.g., material properties)
        df_dpi_explicit = compute_explicit_gradient_f(pi)
        
        # Implicit derivative via the warp field
        # d(omega_prime)/dpi is the derivative of the rotation
        domega_prime_dpi = compute_rotation_derivative(omega, pi, T_pi)
        
        # Gradient of f with respect to direction omega_prime
        df_domega = compute_directional_gradient_f(omega_prime)
        
        df_dpi_implicit = dot_product(df_domega, domega_prime_dpi)
        
        total_gradient_f = df_dpi_explicit + df_dpi_implicit
        
        # 6. Accumulate gradient estimate
        # Importance weight is 1/p(omega) = Area / num_samples if uniform
        # But since we integrate over the whole sphere, the estimator is:
        # (1/N) * sum( gradient_f ) * Area ? 
        # Standard MC: Integral = Area * E[f]. Gradient = Area * E[dF/dpi].
        gradient_sum += total_gradient_f
        
    return (gradient_sum / num_samples) * area

def rotate_direction(omega, pi, T_pi):
    """
    Rotates omega such that a specific reference point maps to T_pi.
    In Loubet et al., this is often a rotation that aligns the discontinuity.
    For a silhouette, T_pi is the view direction to the edge.
    The rotation R moves the discontinuity to a fixed position in the sample space.
    """
    # Implementation detail: Construct rotation matrix/quaternion that rotates
    # the current discontinuity position (if known from previous frame or 
    # computed via auxiliary ray) to T_pi, or rotates omega relative to T_pi.
    # A common approach is to rotate the coordinate system so that T_pi 
    # becomes the pole or a fixed axis.
    # Here we assume a helper that returns the rotated vector.
    # Note: The exact construction of R depends on the geometry.
    # For a sphere, if T_pi is the target, we can rotate omega around the 
    # axis perpendicular to both omega and T_pi by the angle between them?
    # No, R must be a global rotation for the whole integral.
    # Loubet et al. use a rotation that makes the discontinuity stationary.
    # If T(pi) moves, R(pi) is chosen such that R(pi)^-1 * T(pi) = constant.
    # So omega_prime = R(pi) * omega.
    # We need R(pi) such that R(pi) * omega_0 = T(pi).
    # Let omega_0 be a fixed vector (e.g., [0,0,1]).
    # Then R(pi) is the rotation that maps omega_0 to T(pi).
    # Then omega_prime = R(pi) * omega.
    
    # Let's implement this specific mapping:
    omega_0 = Vector3(0, 0, 1)
    # Rotation from omega_0 to T_pi
    axis = cross(omega_0, T_pi)
    angle = acos(dot(omega_0, T_pi))
    R = rotation_matrix(axis, angle)
    
    return R @ omega

def compute_rotation_derivative(omega, pi, T_pi):
    """
    Computes d(R(omega, pi))/dpi.
    This requires differentiating the rotation matrix construction.
    """
    # This is complex and depends on how T_pi is computed.
    # If T_pi is computed via ray tracing, this derivative is not available
    # via standard AD unless the ray tracer is differentiable.
    # [G] In practice, this term is often approximated or computed via
    # finite differences if analytical derivatives are unavailable,
    # or by using differentiable ray tracing libraries.
    # For the purpose of this article, we note it must be computed.
    return Vector3(0, 0, 0) # Placeholder for implementation detail
```

**Parameters:**
- `num_samples`: Number of Monte Carlo samples. Higher values reduce variance.
- `pi`: The scene parameter being optimized.

**Edge Cases and Pitfalls:**
- **Specular Materials**: The method does not support perfectly specular materials because the discontinuity (specular lobe) is a Dirac delta function, which cannot be smoothly reparameterized [S2, unofficial].
- **Degenerate Light Sources**: Light sources with Dirac delta functions (e.g., point lights) are not supported for the same reason [S2, unofficial].
- **Multiple Discontinuities**: The method struggles with multiple discontinuities within the support of the integrand, as a single rotation cannot stationaryize all of them simultaneously [S2, unofficial].
- **Approximation Error**: The gradient is an approximation. If the rotation $R$ does not perfectly match the motion of $T(\pi)$ (i.e., $\frac{\partial R}{\partial \pi} \neq \frac{\partial T}{\partial \pi}$), bias remains [S2, S5, unofficial].
- **Implementation Complexity**: Constructing the correct rotation field $R$ requires tracing auxiliary rays or computing geometric derivatives, which can be non-trivial [S1][S5, unofficial].

## Subtopics

### Spherical Rotation Reparameterization
Spherical Rotation Reparameterization is the specific instantiation of Reparametrized Integral Gradients introduced by Loubet et al. [S5, unofficial]. It applies a spherical rotation $R(\omega, \pi)$ to the integral over the unit sphere $S^2$ to make the discontinuity point $\omega_0$ independent of the parameter $\pi$ [S5, unofficial]. It is chosen when the primary discontinuities are on the sphere (e.g., view-dependent silhouettes or reflections) and the geometry allows for a smooth rotation approximation [S5, unofficial]. It differs from other variants by using rigid body rotations rather than general warping fields [S5, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N)$ per sample, where $N$ is the number of samples. The cost is similar to standard path tracing but includes the overhead of computing the rotation and its derivative [S1][S2, unofficial].
- **Memory Complexity**: $O(1)$ additional memory per sample, assuming the rotation and discontinuity location can be computed on the fly [S1].
- **Performance**: Computationally efficient compared to explicit edge sampling, which requires finding and sampling all edges [S2, unofficial]. However, it may have higher variance if the rotation approximation is poor [S2, unofficial].
- **Strengths**: Eliminates bias from moving discontinuities without explicit edge detection; compatible with standard Monte Carlo estimators; simple to implement for spherical integrals [S1][S5, unofficial].
- **Limitations**: Does not support specular materials or Dirac delta light sources; approximated gradients may be inaccurate if the warp field is not well-matched; difficult to extend to multiple discontinuities [S2, unofficial].
- **When to use**: When optimizing scene parameters that cause smooth motion of silhouettes or reflections, and specular highlights are not critical [S2, S5, unofficial].
- **When not to use**: When the scene contains perfect mirrors or point lights, or when multiple complex discontinuities are present [S2, unofficial].

## Background
Reparametrized Integral Gradients was introduced by Loubet et al. in "Reparameterizing Discontinuous Integrands for Differentiable Rendering" at SIGGRAPH Asia 2019 [S4, S5, unofficial]. It was developed to address the bias in gradients caused by moving visibility discontinuities, which breaks shape reconstruction in inverse rendering [S3]. The method builds on the reparameterization trick, applying it to the rendering equation to remove the dependency of discontinuities on scene parameters [S1][S2, unofficial]. It is widely used in differentiable rendering systems, including Mitsuba 3, for gradient-based optimization [S4, unofficial].

## Sources
[S1] Physics-based differentiable rendering, https://doi.org/10.1145/3388769.3407454, fetched 2026-10-29
[S2] Differentiable Rendering: A Survey, https://doi.org/10.48550/arxiv.2006.12057, fetched 2026-10-26 (unofficial)
[S3] DR.JIT, https://doi.org/10.1145/3528223.3530099, fetched 2026-10-26
[S4] Differentiable ray tracing — from paths to gradients | Bai Liping, https://bailiping.com/differentiable-ray-tracing/, fetched 2026-10-28 (unofficial)
[S5] A Survey on Physics-based Differentiable Rendering - arXiv.org, https://arxiv.org/pdf/2504.01402, fetched 2026-10-29 (unofficial)
