# Utah Approximation (Single Scattering)
Also called: Simple Shading Model

## Overview
The Utah approximation is a shading model for single scattering where external illumination reaches a voxel from a distant source unimpeded by intervening objects or volume absorption [S1, unofficial][S2]. It differs from shadow-aware propagators by ignoring occlusion and attenuation between the light source and the voxel, serving as the foundational step for adding surface-like shading to volume data [S1, unofficial][S2].

## How it works
**Inputs:**
*   Scalar field data $f$ on a regular grid [S1, unofficial][S2].
*   Light direction $\omega'$ (unit vector) [S1, unofficial][S2].
*   View direction $\omega$ (unit vector) [S1, unofficial][S2].
*   Particle albedo $a(X)$ [S1, unofficial][S2].
*   Extinction coefficient $\tau(X)$ [S1, unofficial][S2].
*   Henyey-Greenstein parameter $c$ [S2].
*   Internal emissivity $E(X)$ [S1, unofficial][S2].
*   Incoming illumination $i(X, \omega')$ [S1, unofficial][S2].

**Outputs:**
*   Scattered light $S(X, \omega)$ [S1, unofficial][S2].
*   Total source term $g(X, \omega)$ [S1, unofficial][S2].

**Data Structures:**
*   **Grid:** Regularly spaced 3D array of scalar values $f$ [S2].
*   **Gradient:** $\nabla f$ estimated via central differences [S2].

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
    *   **Henyey-Greenstein Phase Function (Spherical Particles):**
        $$ p(\omega, \omega') = \frac{1}{4\pi} \frac{1-c}{(1 + c^2 - 2cx)^{3/2}} $$
        [S1, unofficial][S2]
        Where $x = \cos \alpha = \omega \cdot \omega'$ [S1, unofficial][S2].
    *   **Lambert Diffuse Phase Function (Large Spherical Particles):**
        $$ p(\omega, \omega') = \frac{8}{3\pi} (\sin \alpha + (\pi - \alpha) \cos \alpha) $$
        [S2]
    *   **Simple Lambert Diffuse Shading (Contour Surface):**
        $$ r(X, \omega, \omega') = \max(\nabla f \cdot \omega', 0) $$
        [S2]
        This simulates shading from a contour surface without polygonalization [S2].

3.  **Compute Scattered Light $S(X, \omega)$:**
    $$ S(X, \omega) = r(X, \omega, \omega') i(X, \omega') $$
    [S1, unofficial][S2]

4.  **Compute Total Source Term $g(X, \omega)$:**
    $$ g(X, \omega) = E(X) + S(X, \omega) $$
    [S1, unofficial][S2]

**Parameters:**
*   $c$: Adjustable constant for Henyey-Greenstein phase function. $c > 0$ for forward scattering, $c < 0$ for backward scattering, $c = 0$ for isotropic scattering [S2].

**Edge Cases and Pitfalls:**
*   **Shadows:** The model assumes no shadows. If shadows are required, the transparency of the volume between the light source and point $X$ must be accounted for [S2].
*   **Gradient Estimation:** Central differences require regularly spaced grid data. Interpolation is needed between grid vertices [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_utah_approximation(f, x, h, omega_prime, omega, a, tau, c, E, i):
    """
    Compute the Utah approximation for single scattering at position x.
    
    Args:
        f: Scalar field accessor (callable or grid).
        x: Position vector (3D).
        h: Grid spacing.
        omega_prime: Light direction vector (unit).
        omega: View direction vector (unit).
        a: Particle albedo at x.
        tau: Extinction coefficient at x.
        c: Henyey-Greenstein parameter.
        E: Internal emissivity at x.
        i: Incoming illumination intensity.
        
    Returns:
        g: Total source term.
        grad_f: Estimated gradient vector.
    """
    # Step 1: Estimate gradient using central differences on a regular grid [S2]
    axes = [(1, 0, 0), (0, 1, 0), (0, 0, 1)]
    grad_f = []
    for axis in axes:
        x_plus = tuple(x[k] + h * axis[k] for k in range(3))
        x_minus = tuple(x[k] - h * axis[k] for k in range(3))
        grad_f.append((f(x_plus) - f(x_minus)) / (2 * h))
    # Interpolate between grid vertices if x is not on a grid point [S2]

    # Step 2: Compute BRDF r(X, omega, omega') using Henyey-Greenstein [S1, unofficial][S2]
    x_dot = sum(o * op for o, op in zip(omega, omega_prime))
    denom = 1 + c * c - 2 * c * x_dot
    # Numerical safeguard; mathematically denom > 0 for c in (-1, 1) and x_dot in [-1, 1]
    if denom <= 1e-12:
        denom = 1e-12
    p_hg = (1.0 / (4.0 * math.pi)) * (1 - c) / (denom ** 1.5)
    r = a * tau * p_hg

    # Step 3: Compute scattered light S(X, omega) [S1, unofficial][S2]
    S = r * i

    # Step 4: Compute total source term g(X, omega) [S1, unofficial][S2]
    g = E + S

    return g, grad_f
```

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [S2]
*   **Memory Complexity:** Not given in the sources. [S2]
*   **Strengths:** Provides surface-like shading without explicit surface extraction [S2].
*   **Limitations:** Ignores shadows and volume absorption between the light source and the voxel [S2].
*   **When to use:** As a baseline shading model before adding shadow and absorption effects [S2].
*   **When not to use:** Not given in the sources. [S2]

## Background
The Utah approximation is named after early shaded images from the University of Utah [S1, unofficial][S2]. It is described in the paper "Optical Models for Direct Volume Rendering" by Marc Levoy [S1, unofficial][S2]. The model is a simplification of single scattering that assumes external illumination reaches the voxel unimpeded [S1, unofficial][S2].

## Sources
[S1] Optical Models for Direct Volume Rendering, https://courses.cs.duke.edu/spring03/cps296.8/papers/max95opticalModelsForDirectVolumeRendering.pdf, fetched 2026-10-26 (unofficial)
[S2] Optical models for direct volume rendering, https://www.academia.edu/49467168/Optical_models_for_direct_volume_rendering, fetched 2026-10-27
