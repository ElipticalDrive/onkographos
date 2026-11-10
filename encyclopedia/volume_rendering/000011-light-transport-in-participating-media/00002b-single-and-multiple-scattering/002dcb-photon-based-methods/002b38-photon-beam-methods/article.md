# Photon Beam Methods

Also called: Virtual Particle Beams, Krueger’s Transport Theory Model

## Overview
Photon Beam Methods render participating media by tracing continuous "beams" of virtual particles through the volume, treating the medium as a generalized ray tracer where interaction laws are defined by optical physics [S2, unofficial]. Unlike discrete photon tracing which samples individual scattering events, this approach integrates the Radiative Transfer Equation (RTE) along continuous beam trajectories, allowing for efficient density estimation and handling of complex scattering patterns [S1][S2, unofficial]. It distinguishes itself from [Photon-based Estimation](2521-photon-based-estimation.md) by using extended beam segments rather than point interactions, and from [Instant Radiosity / Virtual Point Lights (VPLs)](9319-instant-radiosity-virtual-point-lights-vpls.md) by maintaining the continuous spatial structure of light transport rather than discretizing it into point sources.

## How it works
**Inputs and Outputs**
*   **Inputs:** A 3D volume dataset (density, absorption $\sigma_a$, scattering $\sigma_s$), light source positions and intensities, camera parameters, and user-defined virtual particle properties (e.g., photon-like behavior) [S2, unofficial].
*   **Outputs:** A 2D image where pixel intensity represents the integrated radiance along the view ray [S2, unofficial].

**Data Structures**
*   **Beam Segment:** A continuous path defined by a start point $\mathbf{p}_{start}$, end point $\mathbf{p}_{end}$, and direction $\boldsymbol{\omega}$.
*   **Interaction Law:** A function mapping beam position and direction to scattering/absorption coefficients and phase functions [S2, unofficial].

**Steps**
1.  **Beam Generation:** Send a beam of "virtual particles" through the volume from the light source or eye [S2, unofficial]. The user selects the properties of these particles (e.g., photons) and the laws of interaction [S2, unofficial].
2.  **Tracing:** Trace the beam through the volume. At each point $\mathbf{p}$ along the beam, evaluate the local medium properties [S2, unofficial].
3.  **Intensity Calculation:** Compute the intensity contribution at the image plane. The intensity $I$ at a pixel is a path integral along the view ray [S2, unofficial]:
    $$ I = \int_{p_{near}}^{p_{far}} \frac{a(\mathbf{p}) + \rho_{sc}(\mathbf{p}) Q(\mathbf{p})}{e^{\tau(\mathbf{p})}} d\mathbf{p} $$
    where:
    *   $a(\mathbf{p})$ is the absorption coefficient [S2, unofficial].
    *   $\rho_{sc}(\mathbf{p})$ is the scattering coefficient [S2, unofficial].
    *   $Q(\mathbf{p})$ is the generalized source term [S2, unofficial].
    *   $\tau(\mathbf{p})$ is the optical depth from $\mathbf{p}$ to the eye [S2, unofficial].
    *   $p_{near}$ and $p_{far}$ are the near and far clipping planes [S2, unofficial].
4.  **Generalized Source:** The source term $Q(\mathbf{p})$ combines emission and in-scattering [S2, unofficial]:
    $$ Q(\mathbf{p}) = q(\mathbf{p}) + \int_{\Omega} \frac{\rho_{sc}(\mathbf{p})}{\rho_{sc}} I(\mathbf{S}, \boldsymbol{\omega}') d\boldsymbol{\omega}' $$
    where $q(\mathbf{p})$ is the local emission, and the integral represents incoming intensity from all directions $\boldsymbol{\omega}'$ scaled by the scattering phase $\rho_{sc}$ [S2, unofficial].
5.  **Approximation:** Typically, a low albedo approximation is used to simplify the integral in $Q(\mathbf{p})$, reducing it to a sum over all light sources [S2, unofficial].
6.  **Update:** Whenever the beam (or its scattered components) hits the image plane (receiver), update the corresponding pixel [S1].
7.  **Refinement:** Use a small number of beams for a coarse solution. For higher accuracy, re-scale the energy of already traced beams and shoot more beams [S1].

**Pitfalls**
*   **Low Albedo Assumption:** The simplification of $Q(\mathbf{p})$ to a sum over light sources introduces bias in highly scattering media (high albedo) [S2, unofficial].
*   **Virtual Particle Definition:** The method's correctness depends on the user correctly defining "virtual particle" properties and interaction laws to match physical optics [S2, unofficial].

## Subtopics

### [Transient Photon Beams](3296-transient-photon-beams.md)
Transient Photon Beams extends the steady-state method to the transient state by accounting for photon time-of-flight along continuous beam trajectories. It solves the time-resolved RTE using 2D density estimation to mitigate temporal variance. Unlike [Photon Beam Diffusion](3768-photon-beam-diffusion.md), it preserves full directional and temporal structure, and unlike [Photon Marching](4775-photon-marching.md), it uses continuous beam segments.

### [Photon Beam Diffusion](3768-photon-beam-diffusion.md)
Photon Beam Diffusion (PBD) is a hybrid Monte Carlo method for subsurface scattering that interprets incident light as a continuous beam and solves the diffusion equation via numerical integration. It eliminates the quantization and sum-of-Gaussians approximations of Quantized Diffusion (QD), achieving similar quality with 3–5 samples instead of 20–60.

### [Photon Marching](4775-photon-marching.md)
Photon Marching generates "discrete photon beams" by depositing photons at uniform intervals along rays from light sources. It replaces stochastic scattering sampling with deterministic Riemann sum integration along the ray path, distributing a single photon's power among multiple deposited photons at fixed steps.

### Diffuse Single Scattering Integration
An explicit calculation of the diffuse single-scattering term using Monte Carlo integration, replacing the multi-scattering Green's function with a single-scattering response [S3, unofficial]. The response $r^{(1)}$ is calculated as:
$$ r^{(1)}(\vec{x}, \vec{x}_r(t)) = \frac{f_s(\vec{\omega} - \vec{x}_r(t)) e^{-t_0 d_r(t)} F_t(\theta_o, \eta) F_t(\theta_i, 1/\rho) \cos \theta_o}{d_r^2(t)} $$
where $\vec{x}_r(t)$ is the direction and $d_r(t)$ is the distance from $\vec{x}_r(t)$ to $\vec{x}$, $\theta_o = \arccos(\vec{x}_r(t) - \vec{x} \cdot \vec{n})$, and $\theta_i = \arccos(\vec{\omega} \cdot \vec{n})$ [S3, unofficial]. If the surface separates volumes with different indices of refraction, a separate sample sequence starting from the critical angle is used to ensure a smooth profile [S3, unofficial].

### Unbiased Photon Gathering
An unbiased method by Qin et al. that combines each photon's path with the camera path into one path, considering all Monte Carlo factors such as visibility and probability to remove bias [S4].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** Using a small number of bundles allows for a coarse solution quickly; accuracy is improved by re-scaling energy and shooting more bundles [S1].
*   **Strengths:** Can describe various existing volume rendering models as special cases; supports isotropic and anisotropic scattering (with storage scheme changes) [S1][S2, unofficial].
*   **Limitations:** The low albedo approximation limits accuracy in highly scattering media [S2, unofficial].

## Background
The method is based on a transport theory model of particle transfer in inhomogeneous media proposed by Krueger [S2, unofficial]. Krueger showed that existing volume rendering models can be described as special cases of this underlying model, where a beam of "virtual" particles is sent through the volume with user-selected properties and interaction laws [S2, unofficial]. If these particles have photon properties and interaction laws are governed by optical laws, the model becomes a generalized ray tracer [S2, unofficial]. Roysam et al. applied this to the problem of visibility of lighted exit signs through fire-caused non-uniform smoke using a parallelized Monte Carlo particle tracing method [S1].

## Sources
[S1] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
[S2] PDF Volume Visualization: Principles and Advances, https://courses.cs.duke.edu/spring03/cps296.8/papers/KaufmanVolumeVisualization.pdf, fetched 2026-10-28 (unofficial)
[S3] Photon Beam Diffusion for Rendering, https://www.scribd.com/document/255860491/Photon-Beam-Diffusion, fetched 2026-10-29 (unofficial)
[S4] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
