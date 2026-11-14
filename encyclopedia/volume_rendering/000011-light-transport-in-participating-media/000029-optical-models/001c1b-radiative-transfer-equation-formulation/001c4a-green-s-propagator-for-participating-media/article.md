# Green's Propagator for Participating Media

Also called: Propagator G, Radiative Transfer Propagator

## Overview
The Green's propagator $G$ is the fundamental solution to the homogeneous radiative transfer equation, representing the radiance at a point $\mathbf{x}$ in direction $\vec{\omega}$ resulting from a point source at $\mathbf{x}'$ emitting in direction $\vec{\omega}'$ [S1]. It serves as the kernel for light transport in participating media, generalizing the concept of ray tracing with attenuation to include scattering [S1]. Unlike the [Standard volume rendering equation](1165-standard-volume-rendering-equation.md) which integrates emission and scattering directly, the propagator formulation separates the transport physics (attenuation and propagation) from the source terms, enabling iterative solutions like the Neumann series [S2, unofficial].

## How it works
**Inputs:**
*   $\mathbf{x}$: Current position in the medium.
*   $\vec{\omega}$: Current direction of propagation.
*   $\mathbf{x}'$: Source position.
*   $\vec{\omega}'$: Source direction.
*   $s$: Path length traveled from the source.
*   $c(\mathbf{x})$: Extinction coefficient (absorption + scattering) at position $\mathbf{x}$.
*   $b(\mathbf{x}, \vec{\omega}, \vec{\omega}')$: Scattering phase function and coefficient (scattering term).

**Outputs:**
*   $G(s, \mathbf{x}, \vec{\omega}, \mathbf{x}', \vec{\omega}')$: Radiance at $\mathbf{x}$ in direction $\vec{\omega}$ due to the point source.

**Data Structures:**
The propagator is a mathematical function. In implementation, it is often evaluated along a ray segment. For the no-scatter case, it requires evaluating the extinction coefficient along the path.

**Steps:**
1.  **Initialize Initial Condition:** At path length $s=0$, the propagator is a Dirac delta function representing the point source [S1]:
    $$ G(0, \mathbf{x}, \vec{\omega}, \mathbf{x}', \vec{\omega}') = \delta(\mathbf{x} - \mathbf{x}')\delta(\vec{\omega} - \vec{\omega}') $$
2.  **Evaluate No-Scatter Propagator:** In the absence of scattering ($b=0$), the propagator describes light traveling in a straight line with exponential attenuation [S1]:
    $$ G_{\text{noscatter}}(\mathbf{x}, \vec{\omega}, \mathbf{x}', \vec{\omega}') = \delta(\mathbf{x} - \vec{\omega}s - \mathbf{x}')\delta(\vec{\omega} - \vec{\omega}') \exp\left(-\int_0^s c(\mathbf{x} - \vec{\omega}(s - s')) ds'\right) $$
    Here, the term $\delta(\mathbf{x} - \vec{\omega}s - \mathbf{x}')$ ensures the evaluation point lies on the ray originating from $\mathbf{x}'$ in direction $\vec{\omega}'$. The integral computes the optical depth along the path from the source to $\mathbf{x}$ [S1].
3.  **Include Scattering (General Case):** For media with scattering, $G$ is the solution to the homogeneous RTE. It can be expanded via the Neumann series, where the zeroth-order term is $G_{\text{noscatter}}$ and higher-order terms account for single, double, etc., scattering events [S1]. The general solution is not closed-form for arbitrary media and requires iterative integration or approximation [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def green_propagator_no_scatter(x, omega, x_prime, omega_prime, s, extinction_coeff):
    """
    Computes the Green's propagator for a non-scattering medium.
    
    Args:
        x: Current position (vec3)
        omega: Current direction (vec3, normalized)
        x_prime: Source position (vec3)
        omega_prime: Source direction (vec3, normalized)
        s: Path length from source (float)
        extinction_coeff: Function returning c(x) at any point x
        
    Returns:
        Radiance value (float)
    """
    # Check if current position lies on the ray from source
    # delta(x - omega*s - x') is non-zero only if x == x_prime + omega*s
    # In discrete implementation, this is handled by ray marching steps
    if not is_on_ray(x, x_prime, omega, s):
        return 0.0
    
    # Check direction match
    if not vectors_equal(omega, omega_prime):
        return 0.0
    
    # Compute optical depth integral from source to x
    # Path: x_prime to x_prime + omega*s
    # Parameter t goes from 0 to s
    # Point on ray: p(t) = x_prime + omega * t
    # Integral: int_0^s c(p(t)) dt
    
    optical_depth = 0.0
    num_samples = 100 # Implementation detail for numerical integration
    dt = s / num_samples
    for i in range(num_samples):
        t = (i + 0.5) * dt
        point = x_prime + omega * t
        optical_depth += extinction_coeff(point) * dt
        
    # Attenuation factor
    attenuation = exp(-optical_depth)
    
    return attenuation

def is_on_ray(x, origin, direction, length):
    """
    Checks if point x lies on the ray segment defined by origin, direction, and length.
    """
    expected_x = origin + direction * length
    # Allow small epsilon for floating point comparison
    return distance(x, expected_x) < 1e-6
```

**Parameters:**
*   $c(\mathbf{x})$: Extinction coefficient. Typical values depend on the medium (e.g., fog, smoke). Not given in the sources.
*   $s$: Path length. Determined by the geometry.

**Edge Cases:**
*   **Precision:** The Dirac delta functions require careful handling in discrete implementations. In practice, this is managed by ray marching where the propagator is evaluated only at points along the ray [S1].
*   **Boundaries:** The propagator assumes an infinite or bounded medium with appropriate boundary conditions. The no-scatter formula assumes straight-line propagation, which fails if the medium has boundaries that cause refraction or reflection (not covered in the no-scatter model) [S1].

## Subtopics

### Transport Operator
The Transport Operator $\mathbf{G}$ is an operator that acts on an outgoing radiance field $g$ to produce the incoming radiance field at a point, accounting for attenuation along the ray [S2, unofficial]. It integrates the radiance from the surface or medium boundary to the current point, including attenuation [S2, unofficial].
**Formula:**
$$ <\mathbf{G} g>(\mathbf{x}, \vec{\omega}) := \int_0^{\|\mathbf{x}_S - \mathbf{x}\|} T(\mathbf{x}', \mathbf{x}) g(\mathbf{x}', -\vec{\omega}) \, ds + T(\mathbf{x}_S, \mathbf{x}) g(\mathbf{x}_S, -\vec{\omega}) $$
where $T(\mathbf{x}', \mathbf{x})$ is the transmittance from $\mathbf{x}'$ to $\mathbf{x}$, and $\mathbf{x}_S$ is the surface intersection point [S2, unofficial]. This operator is used in the operator notation of the Volume Rendering Equation: $L_i = \mathbf{G}(L_e + \mathbf{K} L_i)$ [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **No-scatter:** $O(N)$ where $N$ is the number of samples along the ray to compute the optical depth integral. Memory is $O(1)$ per ray.
*   **With scattering:** Complexity depends on the number of scattering events considered. The Neumann series expansion leads to $O(N^k)$ for $k$ scattering events if computed naively, but can be optimized with techniques like photon mapping or path tracing. Not given in the sources for specific complexities.

**Measured Performance:**
*   The no-scatter propagator is equivalent to simple raytracing with fog attenuation, a model popular in computer graphics and part of APIs like OpenGL [S1]. This is very fast and widely supported.

**Strengths and Limitations:**
*   **Strengths:** Provides a clear physical interpretation of light transport. The no-scatter case is computationally efficient and easy to implement [S1]. The operator formulation allows for elegant mathematical treatment of the RTE [S2, unofficial].
*   **Limitations:** The general scattering case is complex and requires iterative solutions or approximations. The no-scatter model ignores scattering effects, which are significant in many participating media [S1].

**When to use:**
*   Use the no-scatter propagator for simple fog, haze, or volumetric effects where scattering is negligible or approximated as absorption [S1].
*   Use the general propagator or its operator formulation for accurate rendering of multiple scattering effects in media like smoke, clouds, or biological tissue [S1][S2, unofficial].

## Background
The Green's propagator formulation originates from the radiative transfer equation, a fundamental equation in physics describing light transport in participating media [S1]. The specific formulation and its relation to raytracing are discussed in "Practical Rendering of Multiple Scattering Effects in Participating Media" by Raviraj et al. [S1]. The operator notation, including the Transport Operator, is detailed in "Volume Rendering Equation (VLTE) 2" [S2, unofficial].

## Sources
[S1] Practical Rendering of Multiple Scattering Effects in Participating Media, https://cseweb.ucsd.edu/~ravir/HRPIEG.pdf, fetched 2026-10-28
[S2] ボリュームレンダリング方程式(Volume Rendering Equation) 2, https://rayspace.xyz/notes/VLTE2/, fetched 2026-10-30 (unofficial)
