# Radiative Transfer Equation Formulation

Also called: Equation of Transfer, Equation of Transfer for Light, Radiation Transport Equation, Radiative Transfer Equation, Time-dependent RTE, Transport Equation

## Overview
The Radiative Transfer Equation (RTE) is an integro-differential equation that governs the behavior of light in participating media, describing the rate of change of radiance as it propagates through a medium that absorbs, emits, and scatters radiation [S1, unofficial][S5, S6]. It serves as the fundamental physical model for light transport in volumes, accounting for extinction (absorption and out-scattering) and augmentation (in-scattering and emission) [S2, unofficial][S5]. Unlike simplified models such as the [Absorption-only model](839-absorption-only-model.md) or [Emission-only Model](1016-emission-only-model.md), the RTE captures the full complexity of light-matter interaction, including the redistribution of energy from all directions into the viewing direction via scattering [S3, unofficial][S6]. It is the generalization of the [Standard volume rendering equation](1165-standard-volume-rendering-equation.md) and [Volume Rendering Equation](2098-volume-rendering-equation.md) to arbitrary media without surface boundaries [S6].

## How it works
The RTE defines the differential change in radiance $L$ at a position $\mathbf{p}$ along a direction $\boldsymbol{\omega}$ as the sum of loss terms (extinction) and gain terms (source).

### Inputs and Outputs
*   **Input**: Radiance field $L(\mathbf{p}, \boldsymbol{\omega})$, medium properties (extinction $\sigma_t$, scattering $\sigma_s$, absorption $\sigma_a$), phase function $f_p$, emitted radiance $L_e$.
*   **Output**: The derivative of radiance with respect to path length, $\frac{dL}{ds}$, or the equilibrium radiance distribution when solved.

### Data Structures
The equation operates on scalar fields defined over 6-dimensional phase space (3 for position $\mathbf{p}$, 2 for direction $\boldsymbol{\omega}$ on the unit sphere, and optionally 1 for frequency/wavelength). In discrete implementations, this is represented as:
*   **Grid**: 3D voxel grid storing $\sigma_t(\mathbf{p})$, $\sigma_s(\mathbf{p})$, $L_e(\mathbf{p})$.
*   **Phase Function**: A function $f_p(\boldsymbol{\omega}, \boldsymbol{\omega}')$ mapping incident direction to outgoing direction, often parameterized by the scattering angle $\theta$ where $\cos \theta = \boldsymbol{\omega} \cdot \boldsymbol{\omega}'$ [S7, unofficial].

### Formula
The differential form of the RTE is:

$$
\boldsymbol{\omega} \cdot \nabla_{\mathbf{p}} L(\mathbf{p}, \boldsymbol{\omega}) = -\sigma_t(\mathbf{p}) L(\mathbf{p}, \boldsymbol{\omega}) + L_s(\mathbf{p}, \boldsymbol{\omega})
$$

Where:
*   $\boldsymbol{\omega} \cdot \nabla_{\mathbf{p}}$ is the directional derivative along the ray [S5][S8, unofficial].
*   $\sigma_t(\mathbf{p}) = \sigma_a(\mathbf{p}) + \sigma_s(\mathbf{p})$ is the total extinction coefficient [S3, unofficial][S5][S7, unofficial].
*   $L_s(\mathbf{p}, \boldsymbol{\omega})$ is the source term (augmentation) [S5].

The source term $L_s$ combines emission and in-scattering:

$$
L_s(\mathbf{p}, \boldsymbol{\omega}) = \sigma_a(\mathbf{p}) L_e(\mathbf{p}, \boldsymbol{\omega}) + \sigma_s(\mathbf{p}) \int_{\mathbb{S}^2} f_p(\mathbf{p}, \boldsymbol{\omega}, \boldsymbol{\omega}') L(\mathbf{p}, \boldsymbol{\omega}') d\boldsymbol{\omega}'
$$

Where:
*   $\sigma_a(\mathbf{p}) L_e(\mathbf{p}, \boldsymbol{\omega})$ is the emitted radiance [S5, S6]. Note: Some sources define emission via a separate coefficient $\xi$ or $\sigma_a L_e$ [S3, unofficial][S5].
*   $\sigma_s(\mathbf{p}) \int_{\mathbb{S}^2} f_p(\mathbf{p}, \boldsymbol{\omega}, \boldsymbol{\omega}') L(\mathbf{p}, \boldsymbol{\omega}') d\boldsymbol{\omega}'$ is the in-scattering term [S1, unofficial][S5].
*   $f_p(\mathbf{p}, \boldsymbol{\omega}, \boldsymbol{\omega}')$ is the phase function, normalized such that $\int_{\mathbb{S}^2} f_p(\boldsymbol{\omega}, \boldsymbol{\omega}') d\boldsymbol{\omega}' = 1$ [S7, unofficial].
*   $L(\mathbf{p}, \boldsymbol{\omega}')$ is the incident radiance from direction $\boldsymbol{\omega}'$ [S5].

Alternatively, using the source function $L_s^*$ (where $L_s^* = \frac{\sigma_a}{\sigma_t} L_e + \frac{\sigma_s}{\sigma_t} \int f_p L d\boldsymbol{\omega}'$):

$$
\frac{\partial L(\mathbf{p}, \boldsymbol{\omega})}{\partial t} = -\sigma_t(\mathbf{p}, \boldsymbol{\omega}) L_i(\mathbf{p}, -\boldsymbol{\omega}) + \sigma_t(\mathbf{p}, \boldsymbol{\omega}) L_s^*(\mathbf{p}, \boldsymbol{\omega})
$$

This form is used in integral equation derivations [S6].

### Parameters
*   **Radiance $L$**: Units of $\text{W}/(\text{sr} \cdot \text{m}^2)$ [S2, unofficial].
*   **Extinction Coefficient $\sigma_t$**: Units of $\text{m}^{-1}$ [S3, unofficial].
*   **Albedo**: Ratio $\sigma_s / \sigma_t$. An albedo of 1 implies no true absorption [S3, unofficial].
*   **Phase Function $f_p$**: Depends only on the phase angle $\cos \theta = \boldsymbol{\omega} \cdot \boldsymbol{\omega}'$ [S7, unofficial].

### Pseudocode
The following pseudocode computes the local derivative of radiance for a given point and direction, representing one step of solving the differential equation.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_rte_derivative(p, omega, sigma_t, sigma_s, sigma_a, L_e, phase_function, incident_radiance_field):
    """
    Computes the RHS of the RTE: omega . grad_p L(p, omega).
    
    Args:
        p: 3D position vector.
        omega: Unit direction vector.
        sigma_t: Extinction coefficient at p.
        sigma_s: Scattering coefficient at p.
        sigma_a: Absorption coefficient at p.
        L_e: Emitted radiance at p in direction omega.
        phase_function: Function f_p(omega, omega_prime) returning phase value.
        incident_radiance_field: Function L(p, omega_prime) returning incident radiance.
        
    Returns:
        The rate of change of radiance along omega.
    """
    # 1. Extinction term (loss)
    extinction = -sigma_t * incident_radiance_field(p, omega)
    
    # 2. Emission term (gain)
    emission = sigma_a * L_e
    
    # 3. In-scattering term (gain)
    # Integral over the unit sphere S^2
    # In practice, this is approximated by quadrature (e.g., N-flux or Monte Carlo)
    # Here we denote the exact integral
    in_scattering = 0.0
    for omega_prime in sample_sphere():
        # Get incident radiance from direction omega_prime
        L_incident = incident_radiance_field(p, omega_prime)
        # Get phase function value for scattering from omega_prime to omega
        f_p = phase_function(omega, omega_prime)
        # Add contribution: sigma_s * f_p * L_incident * dOmega
        # Note: dOmega is handled by the sampling distribution
        in_scattering += sigma_s * f_p * L_incident
        
    # Total source term
    source = emission + in_scattering
    
    # Net change
    derivative = extinction + source
    
    return derivative
```

### Edge Cases and Pitfalls
*   **No Analytical Solution**: The RTE is an integro-differential equation with no general analytical solution [S5]. Approximations like N-flux theory (discretizing directions) or diffusion approximation (isotropic scattering) are required [S5].
*   **Singularity**: The phase function may be singular in the forward direction (e.g., Henyey-Greenstein with $g \to 1$), requiring careful numerical integration [S7, unofficial].
*   **Boundary Conditions**: For finite rays, the incident radiance $L_i$ includes surface contributions attenuated by transmittance [S6].

## Subtopics

### Green's Propagator for Participating Media
The Green's propagator $G$ is the solution to the homogeneous RTE (source term $Q=0$) with an initial condition defined by a Dirac delta function representing a point source at $\mathbf{x}'$ and direction $\boldsymbol{\omega}'$ [S7, unofficial]. It allows the solution of the RTE to be expressed as a convolution of the initial radiance with $G$ [S7, unofficial].
*   **Formula**: $G(s=0, \mathbf{x}, \boldsymbol{\omega}, \mathbf{x}', \boldsymbol{\omega}') = \delta(\mathbf{x} - \mathbf{x}')\delta(\boldsymbol{\omega} - \boldsymbol{\omega}')$ [S7, unofficial].
*   **No-Scatter Case**: If scattering is absent ($b=0$), the propagator simplifies to straight-line travel with exponential attenuation: $G(\mathbf{x}, \boldsymbol{\omega}, \mathbf{x}', \boldsymbol{\omega}') = \delta(\mathbf{x} - \boldsymbol{\omega}s - \mathbf{x}')\delta(\boldsymbol{\omega} - \boldsymbol{\omega}') \exp\left(-\int_0^s c(\mathbf{x} - \boldsymbol{\omega}(s-s')) ds'\right)$ [S7, unofficial]. This is equivalent to simple raytracing with fog attenuation [S7, unofficial].

## Operating characteristics
*   **Complexity**: The RTE is computationally expensive due to the integral over the sphere ($\mathbb{S}^2$) for in-scattering. Solving it requires significant memory and computation [S2, unofficial].
*   **Scaling**: Performance scales with the number of scattering events and the complexity of the phase function. Single-scattering approximations are common for interactive rendering [S2, unofficial].
*   **Strengths**: Physically accurate; handles arbitrary media, multiple scattering, and complex lighting [S6].
*   **Limitations**: No general analytical solution; requires numerical approximation (N-flux, diffusion, Monte Carlo) [S5]. High computational cost for multiple scattering [S2, unofficial].
*   **When to use**: When physical accuracy is required for participating media like smoke, fog, or subsurface scattering [S2, S3, unofficial].
*   **When not to use**: For vacuum or non-participating media (use standard rendering equation); for real-time applications where single-scattering or emission-absorption approximations are sufficient [S2, S3, unofficial].

## Background
The RTE was defined in its modern form by Subrahmanyan Chandrasekhar in the book *Radiative Transfer* (1950) [S1, unofficial][S5]. It was introduced to computer graphics by Kajiya in 1986 [S3, unofficial][S5]. The equation is based on energy conservation at the macroscopic scale and describes the change of specific intensity (radiance) due to absorption, emission, and scattering [S3, unofficial]. It is widely used in fire dynamics simulation (e.g., Smokeview, FDS) and scientific visualization [S2, unofficial].

## Sources
[S1] From the Radiative Transfer Equation to the Volume Rendering Equation, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/volume-rendering-summary-equations.html, fetched 2026-10-25 (unofficial)
[S2] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
[S3] Direct Volume Rendering | CGL @ ETHZ, https://cgl.ethz.ch/teaching/former/scivis_07/Notes/stuff/StuttgartCourse/VIS-Modules-06-Direct_Volume_Rendering.pdf, fetched 2026-10-26 (unofficial)
[S4] [Graphics] Volume Rendering - GitHub Pages, https://leeyngdo.github.io/blog/computer-graphics/2024-03-28-volume-rendering/, fetched 2026-10-27 (unofficial)
[S5] 14 Rendering Volume and Subsurface Scattering - Foundations of Visual ..., https://learnvisualcomputing.github.io/rendering-rte.html, fetched 2026-10-27
[S6] Light Transport II: Volume Rendering / The Equation of Transfer, https://pbr-book.org/4ed/Light_Transport_II_Volume_Rendering/The_Equation_of_Transfer.html, fetched 2026-10-27
[S7] PDF Practical Rendering of Multiple Scattering Effects in Participating Media, https://cseweb.ucsd.edu/~ravir/HRPIEG.pdf, fetched 2026-10-28 (unofficial)
[S8] A Null-Scattering Path Integral Formulation of Light Transport, https://www.scribd.com/document/1067625811/A-Null-scattering-Path-Integral-Formulation-of-Light-Transport, fetched 2026-10-29 (unofficial)
