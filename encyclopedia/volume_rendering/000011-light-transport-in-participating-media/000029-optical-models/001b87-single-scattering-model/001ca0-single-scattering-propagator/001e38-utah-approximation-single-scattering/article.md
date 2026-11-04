# Utah Approximation (Single Scattering)
Also called: Simple Shading Model

## Overview
The Utah approximation is a shading model for single scattering where external illumination reaches a voxel from a distant source unimpeded by intervening objects or volume absorption [S1, unofficial][S2]. It differs from shadow-aware propagators by ignoring occlusion and attenuation between the light source and the voxel, serving as the foundational step for adding surface-like shading to volume data [S1, unofficial][S2].

## How it works
**Inputs:**
*   Scalar field data $f$ on a regular grid.
*   Light direction $\omega'$ (unit vector).
*   View direction $\omega$ (unit vector).
*   Particle albedo $a(X)$.
*   Extinction coefficient $\tau(X)$.
*   Phase function parameters (e.g., $c$ for Henyey-Greenstein).
*   Internal emissivity $E(X)$ (optional).

**Outputs:**
*   Scattered light $S(X, \omega)$.
*   Total source term $g(X, \omega)$.

**Data Structures:**
*   **Grid:** Regularly spaced 3D array of scalar values $f$.
*   **Gradient:** $\nabla f$ estimated via central differences.

**Steps:**

1.  **Estimate Surface Normal:**
    Calculate the gradient $\nabla f$ at position $X$ using central differences between regularly spaced gridded data values, then interpolate between grid vertices [S2]. The contour surface normal $N$ is:
    $$ N = \frac{\nabla f}{|\nabla f|} $$
    [S2]

2.  **Compute Bidirectional Reflection Distribution Function (BRDF) $r(X, \omega, \omega')$:**
    Choose one of the following models based on the desired visual effect [S1, unofficial][S2]:

    *   **Particle Scattering:**
        $$ r(X, \omega, \omega') = a(X) \tau(X) p(\omega, \omega') $$
        [S1, unofficial][S2]
        Where $p(\omega, \omega')$ is the phase function.

    *   **Henyey-Greenstein Phase Function (Spherical Particles):**
        $$ p(\omega, \omega') = \frac{1}{4\pi} \frac{1-c}{(1 + c^2 - 2cx)^{3/2}} $$
        [S1, unofficial][S2]
        Where $x = \cos \alpha = \omega \cdot \omega'$ and $c$ is an adjustable constant between -1 and 1 [S2].

    *   **Lambert Diffuse Phase Function (Large Spherical Particles):**
        $$ p(\omega, \omega') = \frac{8}{3\pi} (\sin \alpha + (\pi - \alpha) \cos \alpha) $$
        [S2]
        Where $\alpha = \arccos(\omega \cdot \omega')$.

    *   **Simple Lambert Diffuse Shading (Contour Surface):**
        $$ r(X, \omega, \omega') = \max(\nabla f \cdot \omega', 0) $$
        [S2]
        This simulates shading from a contour surface without polygonalization [S2].

3.  **Compute Scattered Light $S(X, \omega)$:**
    $$ S(X, \omega) = r(X, \omega, \omega') i(X, \omega') $$
    [S1, unofficial][S2]
    Where $i(X, \omega')$ is the incoming illumination from direction $\omega'$ [S1, unofficial][S2].

4.  **Compute Total Source Term $g(X, \omega)$:**
    $$ g(X, \omega) = E(X) + S(X, \omega) $$
    [S1, unofficial][S2]
    Where $E(X)$ is internal emissivity [S1, unofficial][S2].

**Parameters:**
*   $c$: Adjustable constant for Henyey-Greenstein phase function. $c > 0$ for forward scattering, $c < 0$ for backward scattering, $c = 0$ for isotropic scattering [S2].

**Edge Cases and Pitfalls:**
*   **Shadows:** The model assumes no shadows. If shadows are required, the transparency of the volume between the light source and point $X$ must be accounted for [S2].
*   **Gradient Estimation:** Central differences require regularly spaced grid data. Interpolation is needed between grid vertices [S2].
*   **Degenerate Input:** If $\nabla f = 0$, the normal $N$ is undefined. The simple Lambert formula $\max(\nabla f \cdot \omega', 0)$ handles this gracefully by yielding 0.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_utah_approximation(f, x, omega_prime, omega, a, tau, c, E):
    """
    Compute the Utah approximation for single scattering at position x.
    
    Args:
        f: Scalar field function or grid.
        x: Position vector (3D).
        omega_prime: Light direction vector (unit).
        omega: View direction vector (unit).
        a: Particle albedo at x.
        tau: Extinction coefficient at x.
        c: Henyey-Greenstein parameter.
        E: Internal emissivity at x.
        
    Returns:
        g: Total source term.
    """
    # Step 1: Estimate gradient using central differences [S2]
    # Assuming f is a grid function accessible by index or continuous
    # For a grid, central differences are:
    # df/dx = (f(x+dx) - f(x-dx)) / (2*dx)
    # This requires access to neighboring grid points.
    # For simplicity, assume a function grad_f(x) is available or implemented.
    grad_f = estimate_gradient_central_diff(f, x)
    
    # Step 2: Compute BRDF r(X, omega, omega') [S1, unofficial][S2]
    # Option A: Particle scattering with Henyey-Greenstein phase function [S1, unofficial][S2]
    x_dot = dot(omega, omega_prime)
    # Avoid division by zero if denominator is zero (unlikely with c in (-1,1))
    denom = 1 + c*c - 2*c*x_dot
    if denom <= 0:
        denom = 1e-9 # Guard against numerical issues
    p_hg = (1.0 / (4.0 * pi)) * (1 - c) / (denom ** 1.5)
    r = a * tau * p_hg
    
    # Option B: Simple Lambert diffuse shading [S2]
    # r = max(dot(grad_f, omega_prime), 0)
    
    # Step 3: Compute scattered light S(X, omega) [S1, unofficial][S2]
    # i(X, omega_prime) is incoming illumination. 
    # In the simplest Utah approximation, this is often assumed to be constant 
    # or provided by the light source intensity.
    # Let's assume i is a function or constant.
    i = get_incoming_illumination(omega_prime) # Placeholder for light source intensity
    
    S = r * i
    
    # Step 4: Compute total source term g(X, omega) [S1, unofficial][S2]
    g = E + S
    
    return g

def estimate_gradient_central_diff(f, x):
    """
    Estimate gradient using central differences.
    This is a placeholder for the actual implementation which depends on the data structure.
    [S2]
    """
    # Implementation depends on whether f is a grid or a continuous function.
    # For a grid with spacing h:
    # grad_f = [ (f(x + h*e_i) - f(x - h*e_i)) / (2*h) for i in 0..2 ]
    # This is not given in the sources, so we leave it as a comment.
    pass
```

## Operating characteristics
*   **Time Complexity:** $O(1)$ per voxel for shading calculation, assuming gradient estimation is $O(1)$ (central differences on a grid).
*   **Memory Complexity:** $O(1)$ additional memory per voxel for storing intermediate values.
*   **Strengths:** Simple to implement; provides surface-like shading without explicit surface extraction; fast.
*   **Limitations:** Ignores shadows and volume absorption between light and voxel; unrealistic for dense media or complex lighting [S2].
*   **When to use:** When surface-like shading is desired for volume data and performance is critical, or as a base for more complex models.
*   **When not to use:** When accurate lighting, shadows, or volume absorption effects are required [S2].

## Background
The Utah approximation is named after early shaded images from the University of Utah [S1, unofficial][S2]. It is described in the paper "Optical Models for Direct Volume Rendering" by Marc Levoy [S1, unofficial][S2]. The model is a simplification of single scattering that assumes external illumination reaches the voxel unimpeded [S1, unofficial][S2].

## Sources
[S1] Optical Models for Direct Volume Rendering, https://courses.cs.duke.edu/spring03/cps296.8/papers/max95opticalModelsForDirectVolumeRendering.pdf, fetched 2026-10-26 (unofficial)
[S2] Optical models for direct volume rendering, https://www.academia.edu/49467168/Optical_models_for_direct_volume_rendering, fetched 2026-10-27
