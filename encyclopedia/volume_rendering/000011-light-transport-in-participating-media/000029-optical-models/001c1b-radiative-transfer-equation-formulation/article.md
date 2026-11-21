# Radiative Transfer Equation Formulation

Also called: Equation of Transfer, Light transport equation, Radiation Transport Equation, Radiative Transfer Equation (RTE) for Participating Media, Radiative Transport Equation, Time-dependent RTE

## Overview
The Radiative Transfer Equation (RTE) is an integro-differential equation that governs the transport of radiance in participating media, describing how light changes as it propagates through a volume that absorbs, emits, and scatters radiation [S1, S2, unofficial][S3][S4, unofficial][S5][S6, S7, unofficial][S8, S9][S10, book]. It solves the fundamental problem of modeling light transport in arbitrary participating media (such as smoke, fog, clouds, or fire) by accounting for the balance of energy loss (extinction) and energy gain (emission and in-scattering) along a ray path [S1, S2, unofficial][S3][S4, unofficial][S5][S6, S7, unofficial][S8, S9][S10, book]. It differs from the [Volume Rendering Integral](8682-volume-rendering-integral.md) by being the differential form from which the integral is derived, and from the [Light Transport Equation](2098-volume-rendering-equation.md) by explicitly handling volumetric scattering rather than just surface interactions [S10, book][S14].

## How it works
**Inputs and Outputs**
*   **Input:** The radiance field $L(\mathbf{x}, \boldsymbol{\omega})$ at position $\mathbf{x}$ and direction $\boldsymbol{\omega}$, along with medium properties: extinction coefficient $\sigma_t(\mathbf{x})$, scattering coefficient $\sigma_s(\mathbf{x})$, phase function $f_p(\mathbf{x}, \boldsymbol{\omega}', \boldsymbol{\omega})$, and emission term [S1, unofficial][S5][S7, unofficial][S14].
*   **Output:** The directional derivative of radiance $\frac{dL}{ds}$, representing the change in radiance per unit distance traveled in direction $\boldsymbol{\omega}$ [S1, S7, unofficial][S8][S14].

**Data Structures**
The RTE operates on scalar fields defined over 3D space and the unit sphere of directions.
*   **Extinction Coefficient $\sigma_t(\mathbf{x})$:** A scalar field in $\mathbb{R}^3$, units $m^{-1}$ [S1, unofficial][S5][S14]. Defined as $\sigma_t(\mathbf{x}) = \sigma_a(\mathbf{x}) + \sigma_s(\mathbf{x})$, where $\sigma_a$ is absorption and $\sigma_s$ is scattering [S1, unofficial][S5][S12, unofficial].
*   **Phase Function $f_p(\mathbf{x}, \boldsymbol{\omega}', \boldsymbol{\omega})$:** A function mapping incoming direction $\boldsymbol{\omega}'$ and outgoing direction $\boldsymbol{\omega}$ to a probability density, normalized such that $\int_{\mathbb{S}^2} f_p(\mathbf{x}, \boldsymbol{\omega}', \boldsymbol{\omega}) d\boldsymbol{\omega}' = 1$ [S1, unofficial][S14]. Often simplified to depend only on the dot product $\boldsymbol{\omega} \cdot \boldsymbol{\omega}'$ [S5].
*   **Emission:** Defined as $L_e(\mathbf{x}, \boldsymbol{\omega})$ or volume emittance $L_e^V(\mathbf{x}, \boldsymbol{\omega})$ [S7, unofficial][S14].

**Formulas**
The differential form of the RTE describes the change in radiance $L$ along a ray parameterized by distance $s$ in direction $\boldsymbol{\omega}$ [S1, S7, unofficial][S14]:

$$
\frac{d L(\mathbf{x}, \boldsymbol{\omega})}{d s} = -\sigma_t(\mathbf{x}) L(\mathbf{x}, \boldsymbol{\omega}) + \sigma_s(\mathbf{x}) \int_{\mathbb{S}^2} f_p(\mathbf{x}, \boldsymbol{\omega}', \boldsymbol{\omega}) L(\mathbf{x}, \boldsymbol{\omega}') d\boldsymbol{\omega}' + L_e(\mathbf{x}, \boldsymbol{\omega})
$$

Where:
*   $\frac{d L}{d s} = \boldsymbol{\omega} \cdot \nabla L$ is the directional derivative [S1, unofficial][S5, S8].
*   $-\sigma_t(\mathbf{x}) L(\mathbf{x}, \boldsymbol{\omega})$ is the **extinction** term (loss due to absorption and out-scattering) [S1, S7, unofficial][S14].
*   $\sigma_s(\mathbf{x}) \int_{\mathbb{S}^2} f_p(\mathbf{x}, \boldsymbol{\omega}', \boldsymbol{\omega}) L(\mathbf{x}, \boldsymbol{\omega}') d\boldsymbol{\omega}'$ is the **in-scattering** term (gain from other directions) [S1, S7, unofficial][S14].
*   $L_e(\mathbf{x}, \boldsymbol{\omega})$ is the **emission** term (internal light sources) [S7, unofficial][S14].

Alternatively, using the source term $L_s(\mathbf{x}, \boldsymbol{\omega}) = L_e(\mathbf{x}, \boldsymbol{\omega}) + \sigma_s(\mathbf{x}) \int_{\mathbb{S}^2} f_p(\mathbf{x}, \boldsymbol{\omega}', \boldsymbol{\omega}) L(\mathbf{x}, \boldsymbol{\omega}') d\boldsymbol{\omega}'$, the equation is:

$$
(\boldsymbol{\omega} \cdot \nabla) L(\mathbf{x}, \boldsymbol{\omega}) = -\sigma_t(\mathbf{x}) L(\mathbf{x}, \boldsymbol{\omega}) + L_s(\mathbf{x}, \boldsymbol{\omega})
$$

[S1, unofficial][S8][S10, book].

**Steps to Solve (Integral Form)**
To compute the radiance at a point $\mathbf{x}$ along a ray from a surface intersection $\mathbf{x}_S$ in direction $\boldsymbol{\omega}$, the differential equation is integrated using the method of variation of parameters [S14]:

1.  **Calculate Transmittance:** Compute the transmittance $T(\mathbf{x}', \mathbf{x})$ from a point $\mathbf{x}'$ on the ray to the observation point $\mathbf{x}$:
    $$
    T(\mathbf{x}', \mathbf{x}) = \exp \left( -\int_0^{\|\mathbf{x} - \mathbf{x}'\|} \sigma_t(\mathbf{x}' + t\boldsymbol{\omega}) dt \right)
    $$
    [S3][S14].
2.  **Integrate Source Term:** Integrate the source term (emission + in-scattering) weighted by transmittance from the surface intersection $\mathbf{x}_S$ to $\mathbf{x}$:
    $$
    L(\mathbf{x}, \boldsymbol{\omega}) = T(\mathbf{x}_S, \mathbf{x}) L(\mathbf{x}_S, \boldsymbol{\omega}) + \int_0^{\|\mathbf{x} - \mathbf{x}_S\|} T(\mathbf{x}', \mathbf{x}) \left[ L_e(\mathbf{x}', \boldsymbol{\omega}) + \sigma_s(\mathbf{x}') \int_{\mathbb{S}^2} f_p(\mathbf{x}', \boldsymbol{\omega}', \boldsymbol{\omega}) L(\mathbf{x}', \boldsymbol{\omega}') d\boldsymbol{\omega}' \right] ds
    $$
    [S14].
    *Note: The incoming direction for in-scattering is often negated in implementation conventions ($-\boldsymbol{\omega}'$) depending on coordinate system definitions [S14].*

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def directional_derivative_radiance(x, omega, sigma_t, sigma_s, phase_function, L_e, L_field):
    """
    Computes the directional derivative of radiance dL/ds at position x in direction omega.
    Implements the differential form of the RTE.
    """
    # 1. Extinction (Loss)
    extinction = -sigma_t(x) * L_field(x, omega)
    
    # 2. In-scattering (Gain)
    # Integral over the unit sphere S^2 approximated by Monte Carlo
    inscattering = 0.0
    num_samples = 1000
    for _ in range(num_samples):
        # Sample a direction omega_prime uniformly on the unit sphere
        omega_prime = sample_uniform_sphere()
        # Phase function f_p(x, omega', omega)
        f = phase_function(x, omega_prime, omega)
        # Incident radiance from direction omega'
        L_incident = L_field(x, omega_prime)
        # Monte Carlo estimator: integral over sphere is approximated by sum / N * 4pi
        inscattering += sigma_s(x) * f * L_incident * (4.0 * math.pi / num_samples)
        
    # 3. Emission (Gain)
    emission = L_e(x, omega)
    
    # Total derivative dL/ds
    dL_ds = extinction + inscattering + emission
    
    return dL_ds

def compute_transmittance(x_start, x_end, sigma_t_field):
    """
    Computes transmittance T(x_start, x_end) = exp(-integral of sigma_t along path).
    """
    path_vec = (x_end[0]-x_start[0], x_end[1]-x_start[1], x_end[2]-x_start[2])
    path_length = math.sqrt(path_vec[0]**2 + path_vec[1]**2 + path_vec[2]**2)
    if path_length < 1e-6:
        return 1.0
    direction = (path_vec[0]/path_length, path_vec[1]/path_length, path_vec[2]/path_length)
    
    # Numerical integration of extinction along the path
    num_steps = 100
    optical_depth = 0.0
    step_size = path_length / num_steps
    for i in range(num_steps):
        t = (i + 0.5) * step_size
        x_mid = (x_start[0] + t*direction[0], x_start[1] + t*direction[1], x_start[2] + t*direction[2])
        optical_depth += sigma_t_field(x_mid) * step_size
        
    return math.exp(-optical_depth)

def solve_rte_integral(x_obs, x_surface, omega, sigma_t_field, sigma_s_field, phase_function, L_e_field, L_surface, L_field):
    """
    Computes the radiance at x_obs by integrating the RTE along the ray from x_surface.
    Implements the integral form derived from the differential RTE.
    """
    # 1. Calculate Transmittance from surface to observation point
    transmittance_surface = compute_transmittance(x_surface, x_obs, sigma_t_field)
    term_surface = transmittance_surface * L_surface
    
    # 2. Integrate volumetric sources along the path
    path_vec = (x_obs[0]-x_surface[0], x_obs[1]-x_surface[1], x_obs[2]-x_surface[2])
    path_length = math.sqrt(path_vec[0]**2 + path_vec[1]**2 + path_vec[2]**2)
    
    volumetric_contribution = 0.0
    num_steps = 100
    step_size = path_length / num_steps
    for i in range(num_steps):
        t = (i + 0.5) * step_size
        x_prime = (x_surface[0] + t*omega[0], x_surface[1] + t*omega[1], x_surface[2] + t*omega[2])
        
        # Transmittance from x_prime to observation point
        transmittance_vol = compute_transmittance(x_prime, x_obs, sigma_t_field)
        
        # Source term at x_prime: Emission + In-scattering
        emission = L_e_field(x_prime, omega)
        
        # In-scattering integral approximation
        inscattering = 0.0
        num_scatter_samples = 100
        for _ in range(num_scatter_samples):
            omega_prime = sample_uniform_sphere()
            f = phase_function(x_prime, omega_prime, omega)
            L_incident = L_field(x_prime, omega_prime)
            inscattering += sigma_s_field(x_prime) * f * L_incident * (4.0 * math.pi / num_scatter_samples)
            
        source = emission + inscattering
        volumetric_contribution += transmittance_vol * source * step_size
        
    return term_surface + volumetric_contribution

def sample_uniform_sphere():
    """Samples a direction uniformly on the unit sphere S^2."""
    u = random.uniform(0, 1)
    v = random.uniform(0, 1)
    theta = 2.0 * math.pi * u
    phi = math.acos(2.0 * v - 1.0)
    return (math.sin(phi) * math.cos(theta), math.sin(phi) * math.sin(theta), math.cos(phi))
```

**Parameters**
*   $\sigma_t$: Total extinction coefficient ($m^{-1}$) [S1, unofficial][S5][S14].
*   $\sigma_s$: Scattering coefficient ($m^{-1}$) [S1, unofficial][S5][S14].
*   $\sigma_a$: Absorption coefficient ($m^{-1}$), $\sigma_a = \sigma_t - \sigma_s$ [S3][S14].
*   $f_p$: Phase function, normalized to 1 over the sphere [S1, unofficial][S14].
*   $L_e$: Emission coefficient ($W/(sr \cdot m^3)$) [S7, unofficial][S14].
*   $\Lambda$: Scattering albedo, $\Lambda = \sigma_s / \sigma_t$ [S14].

**Edge Cases and Pitfalls**
*   **No Analytical Solution:** The RTE is an integro-differential equation with no general analytical solution for arbitrary media [S8]. Numerical approximations (e.g., diffusion approximation, N-flux theory) are required [S8].
*   **Neglected Physics:** The RTE is based on geometrical optics and neglects diffraction, interference, wave character, and polarization [S2, S6, unofficial].
*   **High Albedo:** Smoke often has low albedo, allowing scattering terms to be simplified or eliminated for interactive rates [S4, unofficial]. High albedo media require full multiple scattering solutions, which are computationally expensive [S1, unofficial][S5].
*   **Coordinate Conventions:** The direction of incoming light in the phase function integral may be negated ($-\boldsymbol{\omega}'$) depending on the specific implementation's coordinate system [S14].

## Subtopics

### [Standard volume rendering equation](1165-standard-volume-rendering-equation.md)
The Standard volume rendering equation computes the final pixel color by integrating emitted and scattered light along a camera ray, attenuated by extinction. It differs from the general RTE by typically assuming no explicit light transport simulation (in-scattering from other light sources) in its basic form, relying instead on direct emission or ambient approximation.

### [Volume Rendering Equation](2098-volume-rendering-equation.md)
The Volume Rendering Equation explicitly formulates the radiative transfer differential equation, allowing for the construction of emission and absorption fields directly from particle distributions or volume grids. Unlike the Standard equation, it handles the full RTE structure, enabling physically plausible gas cloud appearance from continuous density fields.

### [Similarity Theory / High-Order Similarity Relations](10236-similarity-theory-high-order-similarity-relations.md)
Similarity Theory identifies equivalence classes within the RTE parameter space, where distinct sets of scattering parameters produce identical radiance fields if band-limited in the spherical harmonics domain. It enables replacing difficult-to-simulate scattering parameters (e.g., highly forward-scattering) with equivalent, easier-to-simulate ones, accelerating forward rendering.

### [Green's Propagator for Participating Media](1222-green-s-propagator-for-participating-media.md)
The Green's propagator $G$ is the solution to the homogeneous RTE (source term $Q=0$) with an initial condition defined by a Dirac delta function representing a point source at position $\mathbf{x}'$ and direction $\boldsymbol{\omega}'$. In the absence of scattering ($b=0$), the propagator models light traveling in a straight line and attenuated by the absorption coefficient $c(\mathbf{x})$:
$$ G(s=0, \mathbf{x}, \boldsymbol{\omega}, \mathbf{x}', \boldsymbol{\omega}') = \delta(\mathbf{x} - \mathbf{x}')\delta(\boldsymbol{\omega} - \boldsymbol{\omega}') $$
$$ G(\mathbf{x}, \boldsymbol{\omega}, \mathbf{x}', \boldsymbol{\omega}') = \delta(\mathbf{x} - \boldsymbol{\omega}s - \mathbf{x}')\delta(\boldsymbol{\omega} - \boldsymbol{\omega}') \exp\left(-\int_0^s c(\mathbf{x} - \boldsymbol{\omega}(s-s')) ds'\right) $$
.

## Operating characteristics
*   **Complexity:** The RTE is an integro-differential equation. It has no general analytical solution for arbitrary media [S8].
*   **Performance:** Approximations are required for interactive rates. The primary approximation is exploiting low albedo (e.g., in smoke) to simplify or eliminate scattering terms [S4, unofficial].
*   **Scaling:** Computation scales with the number of scattering events and the complexity of the phase function. High-order similarity relations can reduce the number of scattering events required [S1, unofficial].
*   **Strengths:** Physically accurate modeling of light transport in participating media, handling arbitrary absorption, emission, and scattering [S2, unofficial][S3][S4, unofficial][S5][S6, S7, unofficial][S8, S9][S10, book].
*   **Limitations:** Neglects wave effects (diffraction, interference) [S2, S6, unofficial]. Computationally expensive for multiple scattering [S4, unofficial][S8].

## Background
The Radiative Transfer Equation was defined in its modern form by Subrahmanyan Chandrasekhar in the book *Radiative Transfer*, published in 1950 [S1, unofficial][S5][S7, unofficial][S8]. In computer graphics, the equation of transfer for light was introduced by Kajiya in 1986 [S2, S6, unofficial][S8]. Other early models include the emission-absorption model by Hege (1993) and the density-emitter model by Sabella (1988) [S2, S6, unofficial]. It is the basis for all volume rendering methods involving participating media [S2, unofficial][S8].

## Sources
[S1] From the Radiative Transfer Equation to the Volume Rendering Equation, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/volume-rendering-summary-equations.html, fetched 2026-10-25 (unofficial)
[S2] Direct Volume Rendering Techniques, https://www.scribd.com/document/853597780/VIS-Modules-06-Direct-Volume-Rendering, fetched 2026-10-25 (unofficial)
[S3] Physics-based differentiable rendering, https://doi.org/10.1145/3388769.3407454, fetched 2026-10-29
[S4] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
[S5] High-order similarity relations in radiative transfer, https://doi.org/10.1145/2601097.2601104, fetched 2026-10-26
[S6] [PDF] Direct Volume Rendering | CGL @ ETHZ, https://cgl.ethz.ch/teaching/former/scivis_07/Notes/stuff/StuttgartCourse/VIS-Modules-06-Direct_Volume_Rendering.pdf, fetched 2026-10-26 (unofficial)
[S7] [Graphics] Volume Rendering - GitHub Pages, https://leeyngdo.github.io/blog/computer-graphics/2024-03-28-volume-rendering/, fetched 2026-10-27 (unofficial)
[S8] 14 Rendering Volume and Subsurface Scattering - Foundations of Visual ..., https://learnvisualcomputing.github.io/rendering-rte.html, fetched 2026-10-27
[S9] Light Transport II: Volume Rendering / The Equation of Transfer, https://pbr-book.org/4ed/Light_Transport_II_Volume_Rendering/The_Equation_of_Transfer.html, fetched 2026-10-27
[S10] Light Transport II: Volume Rendering / The Equation of Transfer, https://www.pbr-book.org/3ed-2018/Light_Transport_II_Volume_Rendering/The_Equation_of_Transfer.html, fetched 2026-10-27 (book)
[S11] PDF Practical Rendering of Multiple Scattering Effects in Participating Media, https://cseweb.ucsd.edu/~ravir/HRPIEG.pdf, fetched 2026-10-28
[S12] PDF Real-Time Light Transport in Analytically Integrable Participating Media, https://cgg.mff.cuni.cz/~tomas/publications/pub-thesis-real-time-scattering/thesis.pdf, fetched 2026-10-29 (unofficial)
[S13] A Null-Scattering Path Integral Formulation of Light Transport, https://www.scribd.com/document/1067625811/A-Null-scattering-Path-Integral-Formulation-of-Light-Transport, fetched 2026-10-29 (unofficial)
[S14] ボリュームレンダリング方程式 (Volume Rendering Equation) 1, https://rayspace.xyz/CG/contents/VLTE1/, fetched 2026-10-30
