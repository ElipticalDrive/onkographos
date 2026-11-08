# Beam × Beam 1D estimate

Also called: 1D kernel density estimation, Beam × Beam 1D, Beam × Beam 2D, Beam × Beam 2D1 estimate, Photon Density Estimation, Transient 1D Kernel Density Estimation

## Overview
The Beam × Beam 1D estimate is a radiance estimation technique for participating media that combines a camera subpath with a light subpath by finding nearby endpoints of light subpaths and merging them, resulting in one less vertex than the sum of vertices of the subpaths [S2]. It avoids the expensive 3D convolution required by the 3D variant and the ray marching associated with point-based methods, offering better image quality than photon points while being computationally simpler to implement [S1]. It is a specific case of Photon Density Estimation (PDE) where both the light and camera data are represented as beams, and the spatial blur is performed over a line (1D) [S2, S3].

## How it works
### Inputs and Outputs
**Inputs:**
1.  **Light Subpath:** A sequence of vertices sampled from the light, ending at vertex $\mathbf{a}$ (denoted as $\mathbf{y}_{s-2}$ in [S2]). The ray from $\mathbf{a}$ in direction $\omega_a$ defines the **photon beam**.
2.  **Camera Subpath:** A sequence of vertices sampled from the camera, ending at vertex $\mathbf{d}$ (denoted as $\mathbf{z}_{t-2}$ in [S2]). The ray from $\mathbf{d}$ in direction $\omega_d$ defines the **camera beam**.
3.  **Photon Beam Endpoint:** A point $\mathbf{b}$ sampled along the photon beam ray at distance $s_b$ from $\mathbf{a}$ (denoted as $\mathbf{y}_{s-1}$ in [S2]).
4.  **Camera Beam Endpoint:** A point $\mathbf{c}$ sampled along the camera beam ray at distance $s_c$ from $\mathbf{d}$ (denoted as $\mathbf{z}_{t-1}$ in [S2]).
5.  **Beam Properties:**
    *   $\Phi_b$: Photon beam power/flux.
    *   $\rho(\theta_b)$: Phase function value for the beam direction.
    *   $\mu_s$: Scattering coefficient of the medium.
    *   $t_b$: Time of flight associated with the photon beam.
    *   $R_b$: Kernel radius for the 1D spatial blur.
    *   $\theta_{ad}$: Angle between the photon beam direction $\omega_a$ and the camera beam direction $\omega_d$.

**Outputs:**
*   Estimated radiance $L$ at the connection point (or along the ray) for a specific time $t$.

### Data Structures
*   **Photon Beam:** Defined by origin $\mathbf{a}$, direction $\omega_a$, and stored properties ($\Phi_b, \rho, \mu_s, t_b$).
*   **Camera Beam:** Defined by origin $\mathbf{d}$, direction $\omega_d$.
*   **Kernel:** A normalized 1-dimensional kernel function $K_1(\mathbf{b}, \mathbf{c})$ or $K_{1D}(R_b)$.

### Numbered Steps
1.  **Sample Light Subpath:** Generate a light subpath up to vertex $\mathbf{a}$. Define the photon beam ray originating at $\mathbf{a}$ with direction $\omega_a$.
2.  **Sample Camera Subpath:** Generate a camera subpath up to vertex $\mathbf{d}$. Define the camera beam ray originating at $\mathbf{d}$ with direction $\omega_d$.
3.  **Find Nearby Endpoints:** Identify a photon beam endpoint $\mathbf{b}$ on the photon beam ray and a camera beam endpoint $\mathbf{c}$ on the camera beam ray. These points are considered "nearby" if they fall within the kernel support.
4.  **Calculate Geometric Distances:**
    *   Calculate the distance $s_b$ from $\mathbf{a}$ to $\mathbf{b}$.
    *   Calculate the distance $s_c$ from $\mathbf{d}$ to $\mathbf{c}$.
    *   Calculate the distance between $\mathbf{b}$ and $\mathbf{c}$. In the 1D case, the spatial blur is performed over the line connecting the beams. The distance metric $R_b$ typically refers to the perpendicular distance or the distance along the ray projection, depending on the specific kernel implementation. [S3] defines $K_{1D}(R_b)$ where $R_b$ is the spatial blur radius.
5.  **Evaluate Transmittance:**
    *   Calculate transmittance $T_r(\mathbf{a}, \mathbf{b})$ from $\mathbf{a}$ to $\mathbf{b}$.
    *   Calculate transmittance $T_r(\mathbf{c}, \mathbf{d})$ from $\mathbf{c}$ to $\mathbf{d}$.
6.  **Evaluate Kernel:** Compute the 1D kernel value $K_1(\mathbf{b}, \mathbf{c})$ based on the distance between $\mathbf{b}$ and $\mathbf{c}$.
7.  **Compute PDF:** Calculate the probability density function (PDF) for the sampling of the distances $s_b$ and $s_c$. Let $p_{\tau,s-1}^{\rightarrow}(\bar{\mathbf{y}})$ be the PDF for the light subpath distance sampling and $p_{\tau,t-1}^{\rightarrow}(\bar{\mathbf{z}})$ be the PDF for the camera subpath distance sampling.
8.  **Calculate Radiance Estimate:** Combine the terms using the estimator formula.

### Formulas
The radiance estimate $\langle I \rangle_{B-B1D,s}$ for a light subpath with $s$ vertices is given by [S2]:

$$
\langle I \rangle_{B-B1D,s} = T_r(\mathbf{a}, \mathbf{b}) \frac{\rho(\mathbf{b}, \mathbf{c}) K_1(\mathbf{b}, \mathbf{c})}{\sin \theta_{ad}} T_r(\mathbf{c}, \mathbf{d})
$$

Where:
*   $T_r(\mathbf{a}, \mathbf{b})$ is the transmittance from $\mathbf{a}$ to $\mathbf{b}$.
*   $\rho(\mathbf{b}, \mathbf{c})$ is the phase function evaluated at the scattering event from $\mathbf{b}$ to $\mathbf{c}$.
*   $K_1(\mathbf{b}, \mathbf{c})$ is a normalized 1-dimensional kernel.
*   $\theta_{ad}$ is the angle between the photon beam direction $\omega_a$ and the camera beam direction $\omega_d$.
*   $T_r(\mathbf{c}, \mathbf{d})$ is the transmittance from $\mathbf{c}$ to $\mathbf{d}$.

The corresponding PDF $P_{B-B1D,s}$ is given by [S2]:

$$
P_{B-B1D,s} = \frac{\sin \theta_{ad}}{p_{\tau,s-1}^{\rightarrow}(\bar{\mathbf{y}}) K_1(\mathbf{b}, \mathbf{c}) p_{\tau,t-1}^{\rightarrow}(\bar{\mathbf{z}})}
$$

Where:
*   $p_{\tau,s-1}^{\rightarrow}(\bar{\mathbf{y}})$ is the distance PDF with respect to Euclidean length for the light subpath.
*   $p_{\tau,t-1}^{\rightarrow}(\bar{\mathbf{z}})$ is the distance PDF with respect to Euclidean length for the camera subpath.

For the transient case, the temporal function reduces to a Dirac delta function [S3]:

$$
W_{1D}(t - t_b) = \delta(t - t_b)
$$

The full radiance estimate including temporal and beam properties is [S3]:

$$
L_{b|1D}(\mathbf{x}_r, \boldsymbol{\omega}_c, t) = K_{1D}(R_b) \Phi_b \rho(\boldsymbol{\theta}_b) \mu_s \frac{e^{-\mu_s t_b} e^{-\mu_s s_r}}{\sin \theta_b} \delta(t - t_b)
$$

Where:
*   $K_{1D}(R_b)$ is the 1D kernel with radius $R_b$.
*   $\Phi_b$ is the photon beam power.
*   $\rho(\boldsymbol{\theta}_b)$ is the phase function.
*   $\mu_s$ is the scattering coefficient.
*   $t_b$ is the time of the photon beam.
*   $s_r$ is the distance along the ray.
*   $\theta_b$ is the angle related to the beam geometry.

### Parameters
*   **Kernel Radius ($R_b$):** Not given in the sources. Typically determined by a pilot density estimate or photon differentials [S1].
*   **Beam Length:** Long beams are assumed for the estimators [S2].

### Edge Cases and Pitfalls
*   **Beam-Point Estimator:** The Beam-Point estimator is not included in the standard set because it is stated to be too costly to evaluate [S2].
*   **3D Convolution:** The 3D variant is expensive due to 3D convolution requirements [S1]. The 1D variant avoids this.
*   **Over-blurring:** Compared to photon points, the 1D estimate avoids over-blurring and noise associated with point-based reconstructions [S1].
*   **Angle Singularity:** The term $\sin \theta_{ad}$ in the denominator of the PDF and estimator suggests potential instability if the photon and camera beams are parallel ($\theta_{ad} \approx 0$ or $\pi$). The sources do not explicitly state a guard for this, but it is a mathematical implication of the formula [S2].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def beam_beam_1d_estimate(photon_beam, camera_beam, photon_endpoint, camera_endpoint):
    """
    Computes the Beam x Beam 1D radiance estimate.
    
    Args:
        photon_beam: Object containing origin 'a', direction 'omega_a', 
                     power 'Phi_b', phase 'rho', scattering_coeff 'mu_s', time 't_b'.
        camera_beam: Object containing origin 'd', direction 'omega_d'.
        photon_endpoint: Point 'b' on the photon beam ray.
        camera_endpoint: Point 'c' on the camera beam ray.
        
    Returns:
        radiance_estimate: Float value of the radiance estimate.
        pdf: Float value of the probability density function.
    """
    
    # 1. Geometric Setup
    a = photon_beam.origin
    d = camera_beam.origin
    omega_a = photon_beam.direction
    omega_d = camera_beam.direction
    b = photon_endpoint
    c = camera_endpoint
    
    # 2. Distances
    # Distance from a to b
    dist_ab = distance(a, b)
    # Distance from d to c
    dist_dc = distance(d, c)
    
    # 3. Transmittance
    # T_r(a, b)
    # Assuming exponential attenuation with scattering coefficient mu_s
    # Note: Sources use sigma_s or mu_s. We use mu_s as per S3.
    # If absorption is present, it should be included in the extinction coefficient.
    # For simplicity, assuming pure scattering medium or mu_s includes extinction.
    # T_r = exp(-sigma_t * distance). If mu_s is scattering coeff, we need sigma_t.
    # S3 uses e^(-mu_s * t_b) and e^(-mu_s * s_r). This implies mu_s is the extinction coefficient 
    # or the medium is pure scattering and we are tracking scattering events.
    # Let's assume mu_s is the extinction coefficient for transmittance calculation.
    sigma_t = photon_beam.mu_s 
    T_ab = exp(-sigma_t * dist_ab)
    
    # T_r(c, d)
    T_cd = exp(-sigma_t * dist_dc)
    
    # 4. Phase Function
    # rho(b, c) is the phase function for scattering from b to c.
    # This requires the direction from b to c.
    dir_bc = normalize(c - b)
    rho_val = phase_function(dir_bc) # Placeholder for specific phase function evaluation
    
    # 5. Kernel
    # K_1(b, c) is the 1D kernel.
    # The distance metric for 1D kernel is not explicitly defined in terms of vector math 
    # in the sources, but S3 mentions R_b.
    # Typically, for beam-beam, the distance is the perpendicular distance between the rays 
    # or the distance between the sampled points projected onto a common axis.
    # S2 formula uses K_1(b, c).
    # S3 formula uses K_1D(R_b).
    # We will use the distance between b and c as the argument for the kernel, 
    # or a specific 1D projection distance if defined.
    # Given the ambiguity, we use the Euclidean distance as a proxy for the kernel argument 
    # if no specific 1D projection is defined.
    dist_bc = distance(b, c)
    K_1 = kernel_1d(dist_bc, radius=photon_beam.kernel_radius)
    
    # 6. Angle between beams
    # theta_ad is the angle between omega_a and omega_d
    cos_theta = dot(omega_a, omega_d)
    # Clamp to [-1, 1] to avoid numerical issues
    cos_theta = max(-1.0, min(1.0, cos_theta))
    theta_ad = acos(cos_theta)
    sin_theta_ad = sin(theta_ad)
    
    # 7. PDFs
    # p_tau,s-1 and p_tau,t-1 are the distance PDFs.
    # These depend on the sampling strategy.
    # Assuming uniform sampling along the beam for simplicity, or as defined by the path tracer.
    # S2 defines them as p_tau,s-1->(y_bar) and p_tau,t-1->(z_bar).
    # We need the actual PDF values from the sampling process.
    # If not provided, we assume they are known from the path generation.
    p_light = photon_beam.distance_pdf # Placeholder
    p_camera = camera_beam.distance_pdf # Placeholder
    
    # 8. Estimate
    # Check for singularity
    if sin_theta_ad < 1e-6:
        return 0.0, 0.0 # Avoid division by zero
        
    estimate = T_ab * (rho_val * K_1 / sin_theta_ad) * T_cd
    
    # 9. PDF for MIS
    pdf = sin_theta_ad / (p_light * K_1 * p_camera)
    
    return estimate, pdf

def kernel_1d(distance, radius):
    """
    Simple 1D box kernel for demonstration.
    """
    if distance <= radius:
        return 1.0 / (2.0 * radius) # Normalized
    else:
        return 0.0
```

## Subtopics

### Beam × Beam 2D estimate
Uses a 2D kernel for spatial blur. More complex than 1D but potentially more accurate. [Link](4556-beam-beam-2d-estimate.md)

### Beam × Beam 3D estimate
Uses a 3D kernel. Most accurate but computationally expensive due to 3D convolution. [Link](4557-beam-beam-3d-estimate.md)

### Photon Density Estimation (PDE)
The general framework combining camera and light subpaths. Beam × Beam 1D is a specific instance of PDE. [Link](4554-photon-density-estimation.md)

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of photon beams and camera rays. Avoids the $O(N^3)$ or expensive convolution of the 3D variant.
*   **Memory Complexity:** Linear with respect to the number of stored photon beams.
*   **Performance:** Significantly faster than the 3D variant and more efficient than point-based methods for similar image quality, especially with area lights [S1].
*   **Strengths:** Avoids ray marching, better image quality than photon points, simpler implementation than 3D variant.
*   **Limitations:** 1D blur may introduce bias if the beam geometry is complex. Potential singularity when beams are parallel.
*   **When to Use:** When computational efficiency is critical and 3D convolution is too expensive. When better quality than photon points is needed.
*   **When Not to Use:** When high accuracy is required and 3D convolution is feasible. When beams are likely to be parallel.

## Background
The Beam × Beam 1D estimate is part of the Photon Density Estimation (PDE) framework, which combines camera and light subpaths. It was developed as part of the Unified Points, Beams, and Paths (UPBP) method [S2]. The 1D variant was chosen for its implementation simplicity and efficiency compared to the 3D variant [S1]. It builds on the work of Jarosz et al. on kernel density estimation for photon beams [S3].

## Sources
[S1] A comprehensive theory of volumetric radiance estimation using photon points and beams, https://doi.org/10.1145/1899404.1899409, fetched 2026-10-26
[S2] Efficient Hardware Acceleration of Robust Volumetric Light Transport Simulation, https://doi.org/10.1111/cgf.14802, fetched 2026-10-27
[S3] Progressive Transient Photon Beams, https://arxiv.org/abs/1805.09562v1, fetched 2026-10-27
