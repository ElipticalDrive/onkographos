# Beam × Beam 1D estimate

Also called: 1D kernel density estimation, Beam × Beam 1D, Beam × Beam 2D, Beam × Beam 2D1 estimate, Photon Density Estimation, Transient 1D Kernel Density Estimation

## Overview
The Beam × Beam 1D estimate is a radiance estimation technique for participating media that combines a camera beam with a photon beam to estimate in-scattered light without ray marching [S1, S2]. It solves the problem of efficiently computing volumetric light transport by spreading the energy of a photon beam along the camera ray using a 1D kernel, offering better image quality than photon points while being computationally simpler than the 3D variant [S1]. Unlike its sibling [Beam × Beam 2D1 estimate](4776-beam-beam-1d-estimate.md) and the [Beam × Beam 3D estimate](4776-beam-beam-1d-estimate.md), which require 2D or 3D convolutions, the 1D variant restricts spatial blur to a single line, making it the most practical choice for implementation [S1].

## How it works
The method estimates radiance by merging the endpoints of a light subpath (photon beam) and a camera subpath (camera beam). It assumes the use of "long" beams, where the beam represents a continuous trajectory rather than a discrete point [S2]. The core operation is a 1D kernel density estimation along the camera ray [S3].

### Inputs and Outputs
**Inputs:**
1.  **Photon Beam ($b$):** Defined by origin $\mathbf{b}$, direction $\omega_a$, length $L_b$, and total flux $\Phi_b$. The beam represents the light subpath endpoint.
2.  **Camera Beam ($c$):** Defined by origin $\mathbf{c}$, direction $\omega_d$ (view direction), and length $L_c$. The beam represents the camera subpath endpoint.
3.  **Medium Properties:** Scattering coefficient $\mu_s$, phase function $\rho(\mathbf{b}, \mathbf{c})$, and extinction coefficient $\mu_t$ (implied in transmittance).
4.  **Kernel Parameters:** 1D kernel function $K_1$, radius $R_b$.

**Outputs:**
*   Estimated radiance $L$ at the query point on the camera ray.

### Data Structures
*   **Beam:** A structure containing position $\mathbf{p}$, direction $\omega$, length $L$, and flux $\Phi$.
*   **Kernel:** A normalized 1D function $K_1(u)$ where $u$ is the normalized distance along the ray. Common choices include Gaussian or uniform kernels, normalized such that $\int K_1(u) du = 1$.

### Numbered Steps
1.  **Identify Endpoints:** Let $\mathbf{b}$ be the endpoint of the photon beam (light subpath) and $\mathbf{c}$ be the endpoint of the camera beam (camera subpath) [S2].
2.  **Calculate Geometry:**
    *   Compute the distance $d_{bc} = \|\mathbf{b} - \mathbf{c}\|$.
    *   Compute the angle $\theta_{ad}$ between the photon beam direction $\omega_a$ and the camera beam direction $\omega_d$ [S2].
    *   Calculate the transmittance $T_r(\mathbf{b}, \mathbf{c})$ between the two points [S2].
3.  **Evaluate Kernel:**
    *   Determine the projection of the photon beam onto the camera ray. In the 1D estimate, the spatial blur is performed over a line [S3].
    *   The energy is spread on the ray at a single point $\mathbf{r}(s_r)$ from the single point of the beam $\mathbf{b}(s_b)$ [S3].
    *   Evaluate the 1D kernel $K_1$ based on the distance between the beam endpoints and the kernel radius $R_b$ [S3].
4.  **Compute Radiance Estimate:**
    *   Apply the estimator formula (Equation 13 in [S2]) or the transient form (Equation 15 in [S3]).
    *   For the standard steady-state formulation [S2]:
        $$ \langle I \rangle_{B-B1D,s} = T_r(\mathbf{a}, \mathbf{b}) \frac{\rho(\mathbf{b}, \mathbf{c})K_1(\mathbf{b}, \mathbf{c})}{\sin \theta_{ad}} T_r(\mathbf{c}, \mathbf{d}) $$
        where $\mathbf{a}$ and $\mathbf{d}$ are the previous vertices on the light and camera subpaths, respectively [S2].
    *   For the transient formulation [S3]:
        $$ L_{b|1D}(\mathbf{x}_r, \boldsymbol{\omega}_c, t) = K_{1D}(R_b) \Phi_b \rho(\boldsymbol{\theta}_b) \mu_s \frac{e^{-\mu_s t_b} e^{-\mu_s s_r}}{\sin \theta_b} \delta(t - t_b) $$
        where $\delta(t - t_b)$ is the Dirac delta function, indicating the estimate is valid only at the specific time $t_b$ corresponding to the photon beam arrival [S3].
5.  **Weight by PDF:**
    *   Divide the estimator value by the probability density function (PDF) of the sampling process to obtain the unbiased radiance contribution [S2].
    *   The PDF for the B-B1D estimator is:
        $$ P_{B-B1D,s} = \frac{\sin \theta_{ad}}{p_{\tau,s-1}^{\rightarrow}(\bar{\mathbf{y}})K_1(\mathbf{b}, \mathbf{c})p_{\tau,t-1}^{\rightarrow}(\bar{\mathbf{z}})} $$
        where $p_{\tau}$ is the distance PDF with respect to Euclidean length [S2].

### Formulas and Symbols
*   $\langle I \rangle_{B-B1D,s}$: Radiance estimator for the Beam-Beam 1D case with $s$ light subpath vertices [S2].
*   $T_r(\mathbf{u}, \mathbf{v})$: Transmittance between points $\mathbf{u}$ and $\mathbf{v}$ [S2].
*   $\rho(\mathbf{b}, \mathbf{c})$: Phase function value for scattering from direction $\mathbf{b}$ to $\mathbf{c}$ [S2].
*   $K_1(\mathbf{b}, \mathbf{c})$: Normalized 1-dimensional kernel evaluated at the distance between $\mathbf{b}$ and $\mathbf{c}$ [S2].
*   $\theta_{ad}$: Angle between the photon beam direction $\omega_a$ and camera beam direction $\omega_d$ [S2].
*   $p_{\tau,s-1}^{\rightarrow}(\bar{\mathbf{y}})$: PDF of the distance sampled on the light subpath [S2].
*   $p_{\tau,t-1}^{\rightarrow}(\bar{\mathbf{z}})$: PDF of the distance sampled on the camera subpath [S2].
*   $\Phi_b$: Total flux of the photon beam [S3].
*   $\mu_s$: Scattering coefficient [S3].
*   $t_b$: Time of arrival of the photon beam [S3].
*   $\delta(\cdot)$: Dirac delta function [S3].

### Parameters
*   **Kernel Radius ($R_b$):** Determines the spatial extent of the blur. Not given in the sources.
*   **Beam Length:** Assumed to be "long" beams [S2]. Specific lengths depend on the scene and sampling strategy.

### Edge Cases and Pitfalls
*   **Beam-Point Estimator:** The Beam-Point estimator is excluded from consideration because it is stated to be too costly to evaluate [S2].
*   **Singularity:** The term $\sin \theta_{ad}$ in the denominator can approach zero if the photon and camera beams are parallel, potentially causing numerical instability. [G] Implementers should clamp or handle this case.
*   **Temporal Precision:** In the transient formulation, the Dirac delta function $\delta(t - t_b)$ implies that the estimate is non-zero only at the exact arrival time of the photon beam. This requires precise time matching or integration over time bins [S3].
*   **Over-blurring:** While better than photon points, insufficient kernel radius can still lead to blurring artifacts [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def beam_beam_1d_estimate(photon_beam, camera_beam, medium, kernel_func):
    """
    Estimates radiance using the Beam x Beam 1D method.
    
    Args:
        photon_beam: Object with attributes .origin (b), .direction (omega_a), .flux (Phi_b)
        camera_beam: Object with attributes .origin (c), .direction (omega_d)
        medium: Object with attributes .mu_s (scattering coeff), .phase_func()
        kernel_func: Function K_1(u) returning kernel value for distance u
        
    Returns:
        radiance: Estimated radiance value
    """
    b = photon_beam.origin
    c = camera_beam.origin
    omega_a = photon_beam.direction
    omega_d = camera_beam.direction
    
    # 1. Calculate Geometry
    dist_bc = norm(b - c)
    
    # Angle between beams
    cos_theta = dot(omega_a, omega_d)
    sin_theta = sqrt(1 - cos_theta**2)
    
    # Handle singularity
    if sin_theta < 1e-6:
        return 0.0 # Or handle appropriately
        
    theta_ad = acos(cos_theta)
    
    # Transmittance between b and c
    # Assuming homogeneous medium for simplicity, otherwise integrate along segment
    mu_t = medium.extinction_coefficient # Not explicitly in notes, assumed from context
    transmittance_bc = exp(-mu_t * dist_bc)
    
    # Phase function
    phase = medium.phase_func(b, c)
    
    # 2. Evaluate Kernel
    # K_1 is normalized 1D kernel
    kernel_val = kernel_func(dist_bc)
    
    # 3. Compute Estimator (Steady-State Form from [S2])
    # Note: This formula assumes T_r(a,b) and T_r(c,d) are handled by the caller 
    # or passed as context. Here we compute the core B-B1D term.
    # The full estimator includes T_r(a,b) and T_r(c,d) from the subpaths.
    
    # PDF components (simplified, assuming uniform distance sampling for demonstration)
    # p_tau is distance PDF w.r.t Euclidean length. 
    # For a uniform sample over length L, p_tau = 1/L.
    # These are typically provided by the path tracer's sampling context.
    p_tau_light = photon_beam.pdf_distance # Provided by sampler
    p_tau_camera = camera_beam.pdf_distance # Provided by sampler
    
    # Estimator numerator
    estimator_num = transmittance_bc * (phase * kernel_val / sin_theta)
    
    # PDF denominator
    pdf_denom = sin_theta / (p_tau_light * kernel_val * p_tau_camera)
    
    # Radiance contribution
    radiance = estimator_num / pdf_denom
    
    return radiance
```

## Subtopics

### [Steady-State Nonlinear Beam Radiance Estimation](16479-steady-state-nonlinear-beam-radiance-estimation.md)
A technique for estimating radiance along curved beam trajectories in heterogeneous media. It extends the 1D estimate by integrating along numerically computed curved paths rather than straight lines, accounting for spatially varying refractive indices.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of photon beams in the neighborhood of the camera ray, assuming efficient spatial indexing (e.g., BVH or grid) for beam queries. The 1D kernel evaluation is $O(1)$ per beam [S1].
*   **Memory Complexity:** Depends on the storage of photon beams. Each beam requires storing position, direction, length, and flux [S2].
*   **Performance:** The 1D variant is significantly faster than the 3D variant because it avoids expensive 3D convolution [S1]. It provides better image quality than photon points, avoiding over-blurring and noise associated with point-based reconstructions [S1].
*   **Strengths:** Implementation simplicity, good image quality, efficient handling of area light sources [S1].
*   **Limitations:** Assumes straight-line propagation (unlike the nonlinear variant) [S1]. The Beam-Point estimator is too costly and thus not used [S2].

## Background
The Beam × Beam 1D estimate is part of the Photon Density Estimation (PDE) framework for volumetric light transport [S2]. It was introduced as a practical alternative to the more complex 3D Beam × Beam estimate, which requires 3D convolution [S1]. The method builds on the theoretical foundation of photon beams and density estimation in participating media [S1, S2].

## Sources
[S1] A comprehensive theory of volumetric radiance estimation using photon points and beams, https://doi.org/10.1145/1899404.1899409, fetched 2026-10-26
[S2] Efficient Hardware Acceleration of Robust Volumetric Light Transport Simulation, https://doi.org/10.1111/cgf.14802, fetched 2026-10-27
[S3] Progressive Transient Photon Beams, https://arxiv.org/abs/1805.09562v1, fetched 2026-10-27
