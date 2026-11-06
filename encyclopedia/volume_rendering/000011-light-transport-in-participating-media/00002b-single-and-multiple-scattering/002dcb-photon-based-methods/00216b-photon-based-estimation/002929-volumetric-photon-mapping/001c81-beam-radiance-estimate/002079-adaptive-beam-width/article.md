# Adaptive Beam Width

Also called: Photon Beams, Continuous Photon Beams, Beam Radiance Estimation, 2D Beam Radiance Estimate, Varying Kernel Beam Estimate, Jarosz et al. estimate, Photon Beam Rendering with Adaptive Width, Adaptive Beam Width via Photon Differentials

## Overview
Adaptive Beam Width is a radiance estimation technique that models photon energy as continuous beams and integrates their contribution along a camera ray, using photon differentials to determine the beam's cross-sectional size and shape [S1]. It sets itself apart from fixed-width beam methods by adapting the blur kernel to the expected local photon density, thereby avoiding the over-blurring and banding artifacts common in focused regions like volume caustics [S1]. Unlike the 3D beam estimate which averages transmittance through the kernel depth, this method evaluates transmittance at the intersection point and divides by the 2D area, making it more practical for heterogeneous media [S1].

## How it works
**Inputs and Outputs:**
- **Input:** Camera ray origin $\mathbf{x}_c$ and direction $\vec{\omega}_c$; a set of photon beams $B$, where each beam $b$ has a central position/direction, power $\Phi_b$, phase function $f(\theta)$, and differential rays defining its geometry.
- **Output:** Estimated radiance $L_b(\mathbf{x}_c \leftarrow \vec{\omega}_c, s)$ along the camera ray segment $s$.

**Data Structures:**
A photon beam is defined by:
1.  **Central Ray:** Origin $\mathbf{x}$, direction $\vec{\omega}$.
2.  **Differential Rays:** Two auxiliary rays with origins $\mathbf{x}_u, \mathbf{x}_v$ and directions $\vec{\omega}_u, \vec{\omega}_v$. These form a conical frustum with the central ray [S1].
3.  **Heterogeneous Storage (Optional):** For heterogeneous media, store a 1D transmittance function along each beam, analogous to deep shadow maps [S1].

**Parameters:**
-   **Beam Width Factor ($\alpha$):** A global multiplier for the computed beam widths to adjust the tradeoff between banding and blur. Typical value: 3.0 [S1].
-   **Kernel:** Biweight kernel $K(x) = \frac{15}{16}(1 - x^2)^2$ for $x \in [0, 1]$ [S1].

**Algorithm Steps:**

1.  **Compute Beam Cross-Section:**
    For a query point at distance $t$ along the central beam ray $\mathbf{r}(t) = \mathbf{x} + t\vec{\omega}$, compute the positions on the differential rays:
    $$ \mathbf{r}_u(t) = \mathbf{x}_u + t\vec{\omega}_u $$
    $$ \mathbf{r}_v(t) = \mathbf{x}_v + t\vec{\omega}_v $$
    The semi-major and semi-minor axes of the elliptical cross-section are the Euclidean distances:
    $$ a(t) = \|\mathbf{r}(t) - \mathbf{r}_u(t)\| $$
    $$ b(t) = \|\mathbf{r}(t) - \mathbf{r}_v(t)\| $$
    The area of the cross-section is $\mu_R(t) = \pi a(t) b(t)$ [S1].

2.  **Query Photons:**
    Identify all photon beams $b$ that intersect the camera ray's search region. This is typically accelerated using a BVH over all beams [S1].

3.  **Compute Transmittance and Integration:**
    For each photon beam $b$, determine the overlap interval $[t_b^-, t_b^+]$ along the beam and the corresponding interval $[t_c^-, t_c^+]$ along the camera ray.
    
    If the media is homogeneous, the integral of the two transmittance terms can be solved analytically [S1]:
    $$ \int_{t_c^-}^{t_c^+} e^{-\sigma_t t_c} e^{-\sigma_t t_b^c} dt_c = \frac{e^{-\sigma_t (t_c^- - t_c^+)} (|\cos \theta_b| - 1) - 1}{e^{\sigma_t (t_c^- + t_b^-)} \sigma_t (|\cos \theta_b| - 1)} $$
    where $t_b^c = t_b^- - (t_c - t_c^-)|\cos \theta_b|$ is the projected distance along the photon beam, and $\cos \theta_b = \vec{\omega}_c \cdot \vec{\omega}_b$ [S1].
    
    If the media is heterogeneous, evaluate the transmittance $e^{-\sigma_t t_{p,c}}$ at the intersection point using ray marching, and divide by the 2D measure $\pi r^2$ (where $r$ is the effective radius derived from the cross-section) [S1].

4.  **Sum Contributions:**
    Sum the contributions of all intersecting beams. For the 2D estimate (Equation 14 in [S1]):
    $$ L_b(\mathbf{x}_c \leftarrow \vec{\omega}_c, s) \approx \frac{1}{\pi r^2} \sum_{p \in R} f(\theta_p) \Phi_p e^{-\sigma_{t_{p,c}}} $$
    where $r$ is the beam radius at the intersection, scaled by $\alpha$ [S1].

5.  **Handle Singularities:**
    For secondary scattering or area lights, ensure the beam start cross-section is non-zero to avoid division by zero. Use the differential footprint to induce a conical frustum shape [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def estimate_radiance(camera_ray, photon_beams, alpha, sigma_t):
    """
    camera_ray: (origin, direction, length)
    photon_beams: list of Beam objects
    alpha: global width multiplier
    sigma_t: extinction coefficient (homogeneous) or function (heterogeneous)
    """
    radiance = 0.0
    
    # 1. Query intersecting beams using BVH
    intersecting_beams = query_bvh(camera_ray, photon_beams)
    
    for beam in intersecting_beams:
        # 2. Compute cross-section at intersection
        # Assume intersection point t_c along camera ray is known from BVH query
        t_c = camera_ray.origin + camera_ray.direction * intersection_distance
        
        # Compute beam cross-section at the point on the beam closest to t_c
        # Project t_c onto beam direction to find t_b
        cos_theta = dot(camera_ray.direction, beam.direction)
        t_b = dot(t_c - beam.origin, beam.direction) / dot(beam.direction, beam.direction)
        
        # Get differential ray positions at t_b
        r_center = beam.origin + beam.direction * t_b
        r_u = beam.origin_u + beam.direction_u * t_b
        r_v = beam.origin_v + beam.direction_v * t_b
        
        # Semi-axes
        a = norm(r_center - r_u)
        b = norm(r_center - r_v)
        
        # Effective radius for 2D measure, scaled by alpha
        r_eff = alpha * sqrt(a * b)
        area = pi * r_eff * r_eff
        
        # 3. Compute Transmittance
        # For homogeneous media, use analytical integral if needed, 
        # or simple evaluation at intersection for 2D estimate
        if is_homogeneous:
            # Analytical integral of transmittance along overlap
            # Simplified for 2D estimate: evaluate at intersection
            transmittance = exp(-sigma_t * distance(beam.origin, t_c))
        else:
            # Ray march from beam origin to intersection for heterogeneous
            transmittance = ray_march_transmittance(beam.origin, t_c, sigma_t)
            
        # 4. Add contribution
        phase = phase_function(beam.direction, camera_ray.direction)
        radiance += (phase * beam.power * transmittance) / area
        
    return radiance
```

## Subtopics

### Light Source Emission with Differentials
This variant defines the initial geometry of photon beams based on the light source type to ensure tight packing and correct angular coverage [S1].
-   **Singularity Lights (Point/Spot):** Differential ray origins are set to the light source location. Directions are set to cover the angular extent with minimal overlap, where the solid angle per beam is $1/(pdf(\omega) \cdot N)$ [S1].
-   **Area Lights:** The surface is discretized into a jittered grid. Central ray origins are at grid centers, differential origins at grid edges. Directions follow the angular emission PDF [S1].

## Operating characteristics
-   **Time Complexity:** Dominated by the BVH traversal for beam intersection. Current implementations use a BVH over all beams [S1].
-   **Memory Complexity:** Linear in the number of photon beams. Heterogeneous media require storing 1D transmittance functions, which can be compressed [S1].
-   **Performance:** Converges significantly faster than photon points for sharp features. In a volume caustic scene, 32k beams converged in 43 seconds, while 512k photons took nearly twice as long and produced a noisier, over-blurred result [S1].
-   **Strengths:** Eliminates shadow rays by reusing visibility from photon tracing; handles area lights and multiple bounces naturally; avoids singularities via finite cross-sections [S1].
-   **Limitations:** Adaptive width based on differentials works well for single scattering and caustics but is less successful for multiple scattering [S1]. Fixed-width beams cause banding [S1].

## Background
Adaptive Beam Width is part of the photon beam framework introduced by Jarosz et al. [2008] as a generalization of photon mapping [S1]. It builds on the concept of photon differentials from Igehy [1999] and surface-based methods [Schjøth et al. 2007; Fabianowski and Dingliana 2009]. The method was developed to provide a compact, expressive intermediate representation for lighting in participating media, allowing for efficient radiance estimation without shadow rays [S1].

## Sources
[S1] A comprehensive theory of volumetric radiance estimation using photon points and beams, https://doi.org/10.1145/1899404.1899409, fetched 2026-10-26
