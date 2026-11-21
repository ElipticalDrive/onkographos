# Photon Beam Methods

Also called: Photon Beam Tracing, Beam-Based Photon Methods

## Overview
Photon Beam Methods are a family of numerical techniques for rendering participating media and translucent materials that treat incident light as a continuous beam of photons rather than discrete points or rays [S1, unofficial]. Unlike [Photon-based Estimation](2521-photon-based-estimation.md), which typically deposits photons as point sources for density estimation, Photon Beam Methods maintain the directional continuity of light transport, allowing for more efficient integration of scattering profiles along beam trajectories [S1, unofficial]. This approach mitigates variance in complex scenes by combining beam tracing with density estimation or diffusion approximations, avoiding the high computational overhead of standard Monte Carlo sampling for every ray-beam intersection [S1, unofficial].

## Shared principles
The core mechanism of Photon Beam Methods involves two distinct phases: a forward pass that traces light paths from sources to generate a data structure of "photon beams," and a backward pass that queries this structure during rendering.

1.  **Beam Representation**: A photon beam is defined as a continuous directional segment, characterized by its origin $\mathbf{x}_b$, direction $\boldsymbol{\omega}_b$, power $\Phi_b$, and a spatial extent (radius $r$ or length). Unlike discrete photons, the beam stores its entry parameters and direction, allowing it to be treated as an extended source [S1, unofficial].
2.  **Beam Tracing (Pass 1)**: Paths are traced from light sources using Monte Carlo sampling. At each interaction point (scattering event), a new photon beam is generated and stored in a map or list. The beam's power is attenuated by the medium's scattering coefficient $\mu_s$ (or $\sigma_s$) and transmittance. In transient variants, the starting time $t_{b_0}$ is also computed based on the time-of-flight from the light source to the interaction point [S1, unofficial].
3.  **Ray-Beam Intersection (Pass 2)**: During rendering, camera rays are traced. For each camera ray, the algorithm finds all intersecting photon beams from the map. The intersection is typically defined as a spatial overlap (e.g., a ray passing within a beam's radius $r$) or a geometric intersection interval $[s_r^-, s_r^+]$ [S1, unofficial].
4.  **Density Estimation / Integration**: The radiance contribution from a beam is computed by integrating the beam's power over the intersection interval. This often involves a 2D kernel $\Omega_R(r^2)$ to smooth the contribution and reduce variance [S1, unofficial]. In diffusion variants, the beam is treated as a series of weighted monopoles, and the diffusion equation is solved numerically along the beam trajectory [S1, unofficial].

## Subtopics

### [Transient Photon Beams](3296-transient-photon-beams.md)
Transient Photon Beams extends the steady-state method to the transient regime, computing time-resolved radiance $L(\mathbf{x}_r, \boldsymbol{\omega}_r, t)$. It is chosen when simulating light propagation in complex scenes where time-of-flight effects are significant (e.g., through scattering media). It differs from siblings by explicitly tracking photon time-of-flight along continuous beam trajectories and using a 2D density estimator to mitigate variance without Monte Carlo sampling of the temporal integral.

### [Photon Beam Diffusion](3768-photon-beam-diffusion.md)
Photon Beam Diffusion (PBD) is a hybrid method for rendering subsurface scattering in translucent materials. It interprets incident light as a continuous beam and uses Monte Carlo integration to solve the diffusion equation. It is chosen for efficient, accurate rendering of subsurface scattering, particularly at oblique angles. It differs from [Photon-based Estimation](2521-photon-based-estimation.md) and Quantized Diffusion by eliminating quantization errors through extended source integration and using improved diffusion profiles.

### [Photon Marching](4775-photon-marching.md)
Photon Marching is a simplified photon shooting method identical to ray marching with rays originating at lights. Instead of sampling in-scattered light, a photon is deposited at uniform intervals along the ray length, creating a 'discrete photon beam'. It is chosen for its simplicity and speed in homogeneous media, though it lacks the continuous beam integration of other methods in this family.

### Diffuse Single Scattering Integration
This variant explicitly calculates the diffuse single-scattering term using the same Monte Carlo integration strategy as multi-scattering, replacing the multi-scattering Green's function with a single-scattering response [S1, unofficial].
*   **Definition**: The single-scattering profile $R^{(1)}$ is computed using the estimator $f(\vec{x}, \vec{\omega}, t_i) = r^{(1)}(\vec{x}, \vec{x}_r(t_i)) Q(t_i)$ [S1, unofficial].
*   **Formula**: The single-scattering response is given by:
    $$ r^{(1)}(\vec{x}, \vec{x}_r(t)) = \frac{f_s(\vec{\omega} - \vec{x}_r(t)) e^{-t_0 d_r(t)} F_t(\theta_o, \eta) F_t(\theta_i, 1/\eta) \cos \theta_o}{d_r^2(t)} $$
    where $\vec{x}_r(t)$ and $d_r(t)$ are the direction and distance from $\vec{x}_r(t)$ to $\vec{x}$, $\theta_o = \arccos(\vec{x}_r(t) - \vec{x} \cdot \vec{n})$, $\theta_i = \arccos(\vec{\omega} \cdot \vec{n})$, and $F_t$ and $f_s$ are transmissive Fresnel and phase function terms [S1, unofficial].
*   **Pitfall**: If the surface separates volumes with different indices of refraction, a separate sample sequence for single-scattering must start from the critical angle, as contributions below the critical angle are zero. This guarantees a smooth profile regardless of sample count [S1, unofficial].

## Background
Photon Beam Methods originated from the need to efficiently render participating media and translucent materials with high accuracy. The concept of treating light as continuous beams rather than discrete particles was introduced to improve the convergence of Monte Carlo methods in complex scattering scenarios [S1, unofficial]. The specific application to diffusion for subsurface scattering was detailed in "Photon Beam Diffusion for Rendering" [S1, unofficial], which combined beam tracing with the diffusion approximation to achieve faster and more stable results than previous quantized methods. The transient extension further generalized the method to handle time-dependent light transport [S1, unofficial].

## Sources
[S1] Photon Beam Diffusion for Rendering, https://www.scribd.com/document/255860491/Photon-Beam-Diffusion, fetched 2026-10-29 (unofficial)
