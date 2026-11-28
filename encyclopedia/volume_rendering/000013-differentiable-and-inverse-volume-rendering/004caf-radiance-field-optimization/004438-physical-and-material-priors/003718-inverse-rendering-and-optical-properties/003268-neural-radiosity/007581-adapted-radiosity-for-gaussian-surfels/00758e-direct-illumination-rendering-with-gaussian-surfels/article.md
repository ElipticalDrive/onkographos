# Direct Illumination Rendering with Gaussian Surfels

Also called: Forward Rendering with Gaussian Surfels

## Overview
This method computes the outgoing radiance of Gaussian surfels under direct illumination only, treating light sources as special infinitesimal emissive Gaussian surfels. It differs from global illumination variants by ignoring inter-reflections, resulting in a non-recursive equation that can be evaluated directly without iterative solvers [S1]. It builds on [Adapted Radiosity for Gaussian Surfels](23269-adapted-radiosity-for-gaussian-surfels.md) by specializing the framework to direct light transport.

## How it works
**Inputs:**
- Set of Gaussian surfels representing geometry and material.
- Set of light sources (LS), represented as point lights, directional lights (approximated), or environment maps.
- Kernel function $\kappa$ (e.g., Gaussian, Uniform, Dirac delta) with local support $P$.

**Outputs:**
- Outgoing radiosity coefficients $\mathbf{B}_i^c$ for each surfel $i$.

**Data Structures:**
Each Gaussian surfel $i$ stores:
- Center position $\mathbf{p}_i$.
- Opacity $\alpha_i$.
- Surface normal $\mathbf{n}_i$.
- Emission coefficient vector $\mathbf{E}_i^c$ (spherical harmonics coefficients).
- BRDF coefficient vector $\mathbf{f}_i^c$ (spherical harmonics coefficients).
- Kernel integral $\Lambda_i = \int_P \kappa(\mathbf{x}) d\mathbf{x}$.

Light sources $j \in \text{LS}$ are treated as emissive surfels with:
- Position $\mathbf{p}_j$.
- Emission coefficient $\mathbf{E}_j^c$.
- For directional lights: position far away, inverse square falloff disabled.

**Formulas:**
The forward rendering equation for direct illumination is:

$$
\mathbf{B}_i^c = \mathbf{E}_i^c + \frac{\alpha_i(\mathbf{p}_i)}{\Lambda_i} \times \sum_{j \in \text{LS}} \int_{P_j} \mathbf{f}_j(\omega_I) \times \left( \mathbf{Y}(\omega_I)^T \mathbf{E}_j^c \right) \times \mathbf{V}_{ji}(\mathbf{p}_j, \mathbf{x}) d\mathbf{x}
$$

Where:
- $\mathbf{B}_i^c$: Outgoing radiosity coefficient vector for surfel $i$.
- $\mathbf{E}_i^c$: Emission coefficient vector for surfel $i$.
- $\alpha_i(\mathbf{p}_i)$: Opacity of surfel $i$ at its center.
- $\Lambda_i$: Integral of the kernel function $\kappa$ over its support $P$.
- $\text{LS}$: Set of light source indices.
- $\mathbf{f}_j(\omega_I)$: BRDF of surfel $j$ evaluated at incoming direction $\omega_I$.
- $\mathbf{Y}(\omega_I)$: Spherical harmonics basis vector evaluated at $\omega_I$.
- $\mathbf{E}_j^c$: Emission coefficient vector of light source $j$.
- $\mathbf{V}_{ji}(\mathbf{p}_j, \mathbf{x})$: Fused decay component including angular factor, kernel, and visibility.

Intermediate terms:
$$
\omega_I = \frac{\mathbf{x} - \mathbf{p}_j}{\|\mathbf{x} - \mathbf{p}_j\|_2^2}
$$
$$
\mathbf{f}_i(\omega_I) = \mathbf{f}_i^c \times \bar{\mathbf{Y}}(\omega_I)
$$
$$
V_{ji}(\mathbf{p}_j, \mathbf{x}) = \frac{|\mathbf{n}_i \cdot \omega_I|}{\|\mathbf{x} - \mathbf{p}_j\|_2^2} \kappa_i(\mathbf{x}) \prod_{k=1}^{j-1} (1 - \alpha_k)
$$

Where:
- $\omega_I$: Incoming direction from point $\mathbf{x}$ in light source support to surfel $i$.
- $\mathbf{f}_i^c$: BRDF coefficient vector for surfel $i$.
- $\bar{\mathbf{Y}}(\omega_I)$: Conjugate or specific basis transformation for BRDF evaluation [S1].
- $\mathbf{n}_i$: Surface normal of surfel $i$.
- $\kappa_i(\mathbf{x})$: Kernel function value at point $\mathbf{x}$ for surfel $i$.
- $\alpha_k$: Opacity of preceding kernels (visibility term).

**Parameters:**
- Kernel type: Gaussian, Uniform, or Dirac delta [S1].
- Light representation: Point lights are standard; directional lights are approximated by placing point lights far away and disabling inverse square falloff [S1]. Environment maps are handled similarly [S1].

**Pitfalls:**
- The equation is non-recursive, so no iterative solver is needed [S1].
- Gradient calculation is efficient via auto-differentiation [S1].
- Partial understanding of full light transport (direct illumination alone) benefits sparse-view relighting compared to data-driven approaches [S1].
- The method effectively caches radiance within each kernel, enabling view-independent rendering as with classical radiosity theory [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_direct_illumination(surfels, light_sources):
    """
    Compute outgoing radiosity coefficients for direct illumination.
    
    Args:
        surfels: List of surfel objects with attributes:
                 - p_i: center position
                 - alpha_i: opacity
                 - n_i: surface normal
                 - E_i_c: emission coefficient vector
                 - f_i_c: BRDF coefficient vector
                 - Lambda_i: kernel integral
                 - kappa_i: kernel function
        light_sources: List of light source objects with attributes:
                       - p_j: position
                       - E_j_c: emission coefficient vector
                       - is_directional: boolean flag
                       - disable_falloff: boolean flag
    
    Returns:
        B_i_c: List of outgoing radiosity coefficient vectors for each surfel
    """
    B_i_c = []
    
    for surfel in surfels:
        p_i = surfel.p_i
        alpha_i = surfel.alpha_i
        n_i = surfel.n_i
        E_i_c = surfel.E_i_c
        f_i_c = surfel.f_i_c
        Lambda_i = surfel.Lambda_i
        kappa_i = surfel.kappa_i
        
        # Initialize outgoing radiosity with emission
        B_i = E_i_c.copy()
        
        # Sum over all light sources
        for light in light_sources:
            p_j = light.p_j
            E_j_c = light.E_j_c
            
            # Integrate over light source support P_j
            # Note: For point lights, P_j is infinitesimal, so integral simplifies
            # For directional lights, P_j is far away with disabled falloff
            # For environment maps, similar handling
            
            # Compute integral term
            integral_term = 0.0
            
            # For point lights, the integral over P_j collapses to the light position
            # Assuming point light approximation for simplicity
            x = p_j  # Point light position
            
            # Compute incoming direction
            # Note: Formula uses ||x - p_j||_2^2 in denominator, but x == p_j for point light
            # This suggests the integral is over the light's support, not the light position itself
            # For point lights, we evaluate at the light position and apply inverse square falloff
            # unless disabled for directional approximation
            
            if light.disable_falloff:
                dist_sq = 1.0  # Avoid division by zero, effectively infinite distance
            else:
                dist_sq = np.linalg.norm(x - p_j)**2
                if dist_sq == 0:
                    dist_sq = 1e-10  # Guard against division by zero
            
            omega_I = (x - p_j) / dist_sq
            
            # Evaluate BRDF
            # f_i(omega_I) = f_i_c * Y_bar(omega_I)
            # Note: Y_bar is the conjugate or specific basis transformation
            Y_bar_omega_I = evaluate_spherical_harmonics_conjugate(omega_I)
            f_i_omega_I = f_i_c * Y_bar_omega_I
            
            # Evaluate emission term
            Y_omega_I = evaluate_spherical_harmonics_basis(omega_I)
            emission_term = Y_omega_I.T @ E_j_c
            
            # Compute fused decay component V_ji
            # V_ji = |n_i . omega_I| / ||x - p_j||_2^2 * kappa_i(x) * product(1 - alpha_k)
            # Note: product(1 - alpha_k) is visibility from preceding kernels
            # For direct illumination, we assume no occlusion from other kernels for simplicity
            # or compute visibility based on depth ordering
            
            angular_factor = abs(np.dot(n_i, omega_I))
            kernel_value = kappa_i(x)
            
            # Visibility term: product of (1 - alpha_k) for preceding kernels
            # This requires depth sorting and checking occlusion
            # For simplicity, assume no occlusion (visibility = 1)
            visibility = 1.0
            
            V_ji = (angular_factor / dist_sq) * kernel_value * visibility
            
            # Accumulate integral term
            # For point lights, the integral is approximated by the value at the light position
            # multiplied by the light's emission strength
            integral_term += f_i_omega_I * emission_term * V_ji
        
        # Normalize by opacity and kernel integral
        B_i += (alpha_i / Lambda_i) * integral_term
        
        B_i_c.append(B_i)
    
    return B_i_c

def evaluate_spherical_harmonics_basis(omega_I):
    """
    Evaluate spherical harmonics basis vector at direction omega_I.
    
    Args:
        omega_I: Incoming direction vector
    
    Returns:
        Y: Spherical harmonics basis vector
    """
    # Implementation depends on the specific spherical harmonics order used
    # For SH3 (9 coefficients), this would compute the 9 basis functions
    # at the given direction
    # Not given in the sources, so placeholder for implementation
    pass

def evaluate_spherical_harmonics_conjugate(omega_I):
    """
    Evaluate conjugate spherical harmonics basis vector at direction omega_I.
    
    Args:
        omega_I: Incoming direction vector
    
    Returns:
        Y_bar: Conjugate spherical harmonics basis vector
    """
    # Implementation depends on the specific spherical harmonics order used
    # Not given in the sources, so placeholder for implementation
    pass
```

## Subtopics

### [Global Illumination Rendering with Gaussian Surfels](23295-global-illumination-rendering-with-gaussian-surfels.md)
Extends the framework to model global illumination by accounting for inter-reflections, treating light sources as special Gaussian surfels and assuming non-overlapping kernels. Unlike direct illumination, it handles indirect light transport for accurate geometry reconstruction in scenes with rich inter-reflections.

## Operating characteristics
- **Time Complexity:** Non-recursive, allowing direct evaluation without iterative solvers [S1]. Gradient calculation is efficient via auto-differentiation [S1].
- **Memory Complexity:** Depends on the number of surfels and light sources. Each surfel stores coefficients and kernel parameters.
- **Performance:** Effective for sparse-view relighting compared to data-driven approaches [S1]. View-independent rendering as with classical radiosity theory [S1].
- **Strengths:** Direct evaluation, efficient gradients, view-independent rendering, handles various light types (point, directional, environment maps) [S1].
- **Limitations:** Only models direct illumination, ignoring inter-reflections [S1]. Approximations for directional lights and environment maps [S1].
- **When to use:** Scenes with dominant direct illumination, sparse-view relighting, view-independent rendering requirements [S1].
- **When not to use:** Scenes with significant inter-reflections or global illumination effects [S1].

## Background
Originated in the paper "Differentiable Light Transport with Gaussian Surfels via Adapted Radiosity for Efficient Relighting and Geometry Reconstruction" [S1]. The method builds on classical radiosity theory, caching radiance within kernels for view-independent rendering [S1]. It supports optimization of lighting and geometry through differentiable rendering [S1].

## Sources
[S1] Differentiable Light Transport with Gaussian Surfels via Adapted Radiosity for Efficient Relighting and Geometry Reconstruction, https://doi.org/10.1145/3763305, fetched 2026-10-27
