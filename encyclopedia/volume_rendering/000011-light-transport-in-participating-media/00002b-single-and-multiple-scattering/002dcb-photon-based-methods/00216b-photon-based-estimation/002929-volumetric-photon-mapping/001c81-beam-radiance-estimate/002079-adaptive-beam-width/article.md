# Adaptive Beam Width

Also called: Photon Beams, Continuous Photon Beams, Beam Radiance Estimation, 2D Beam Radiance Estimate, Varying Kernel Beam Estimate, Jarosz et al. estimate, Photon Beam Rendering with Adaptive Width, Adaptive Beam Width via Photon Differentials

## Overview
Adaptive Beam Width is a radiance estimation technique that models photon energy as continuous beams and integrates their contribution along a camera ray, using photon differentials to determine the beam's cross-sectional size and shape [S1]. It sets itself apart from fixed-width beam methods by adapting the blur kernel based on the geometric footprint of photon differentials, thereby avoiding the over-blurring and banding artifacts common in focused regions like volume caustics [S1]. Unlike the 3D beam estimate which averages transmittance through the kernel depth, this method evaluates transmittance at the intersection point and divides by the 2D area, making it more practical for heterogeneous media [S1].

## How it works
**Inputs and Outputs:**
- **Input:** Camera ray origin $\mathbf{x}_c$ and direction $\vec{\omega}_c$; a set of photon beams $B$, where each beam $b$ has a central position/direction, power $\Phi_b$, phase function $f(\theta)$, and differential rays defining its geometry [S1].
- **Output:** Estimated radiance $L_b(\mathbf{x}_c \leftarrow \vec{\omega}_c, s)$ along the camera ray segment $s$ [S1].

**Data Structures:**
A photon beam is defined by:
1.  **Central Ray:** Origin $\mathbf{x}$, direction $\vec{\omega}$ [S1].
2.  **Differential Rays:** Two auxiliary rays with origins $\mathbf{x}_u, \mathbf{x}_v$ and directions $\vec{\omega}_u, \vec{\omega}_v$ [S1]. These form a conical frustum with the central ray [S1].
3.  **Heterogeneous Storage (Optional):** For heterogeneous media, store a 1D transmittance function along each beam, analogous to deep shadow maps [S1].

**Parameters:**
-   **Beam Width Factor ($\alpha$):** A global multiplier for the computed beam widths to adjust the tradeoff between banding and blur. Typical value: 3.0 [S1].
-   **Kernel:** Biweight kernel $K(x) = \frac{15}{16}(1 - x^2)^2$ for $x \in [0, 1]$ [S1].

**Algorithm Steps:**
1.  **Compute Beam Cross-Section:** For a query point at distance $t$ along the central beam ray $\mathbf{r}(t) = \mathbf{x} + t\vec{\omega}$, compute the positions on the differential rays: $\mathbf{r}_u(t) = \mathbf{x}_u + t\vec{\omega}_u$ and $\mathbf{r}_v(t) = \mathbf{x}_v + t\vec{\omega}_v$ [S1]. The semi-major and semi-minor axes of the elliptical cross-section are the Euclidean distances: $a(t) = \|\mathbf{r}(t) - \mathbf{r}_u(t)\|$ and $b(t) = \|\mathbf{r}(t) - \mathbf{r}_v(t)\|$ [S1]. The 2D measure for the circular cross-section approximation is $\mu_R(r^2) = \pi r^2$ [S1].
2.  **Query Photons:** Identify all photon beams $b$ that intersect the camera ray's search region [S1]. This is accelerated using a BVH over all beams [S1].
3.  **Compute Transmittance and Integration:** For each photon beam $b$, determine the overlap interval $[t_b^-, t_b^+]$ along the beam and the corresponding interval $[t_c^-, t_c^+]$ along the camera ray [S1]. If the media is homogeneous, the integral of the two transmittance terms can be solved analytically: $\int_{t_c^-}^{t_c^+} e^{-\sigma_t t_c} e^{-\sigma_t t_b^c} dt_c = \frac{e^{-\sigma_t (t_c^- - t_c^+)} (|\cos \theta_b| - 1) - 1}{e^{\sigma_t (t_c^- + t_b^-)} \sigma_t (|\cos \theta_b| - 1)}$ where $t_b^c = t_b^- - (t_c - t_c^-)|\cos \theta_b|$ is the projected distance along the photon beam, and $\cos \theta_b = \vec{\omega}_c \cdot \vec{\omega}_b$ [S1]. If the media is heterogeneous, evaluate the transmittance $e^{-\sigma_t t_{p,c}}$ at the intersection point using ray marching, and divide by the 2D measure $\pi r^2$ [S1].
4.  **Sum Contributions:** Sum the contributions of all intersecting beams. For the 2D estimate: $L_b(\mathbf{x}_c \leftarrow \vec{\omega}_c, s) \approx \frac{1}{\pi r^2} \sum_{p \in R} f(\theta_p) \Phi_p e^{-\sigma_{t_{p,c}}}$ where $r$ is the beam radius at the intersection, scaled by $\alpha$ [S1].
5.  **Handle Singularities:** For secondary scattering or area lights, ensure the beam start cross-section is non-zero to avoid division by zero. Use the differential footprint to induce a conical frustum shape [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def estimate_radiance(camera_origin, camera_dir, photon_beams, alpha, sigma_t_func):
    """
    camera_origin: (x, y, z)
    camera_dir: (dx, dy, dz) normalized
    photon_beams: list of dicts with keys 'origin', 'dir', 'origin_u', 'dir_u', 'origin_v', 'dir_v', 'power'
    alpha: float (global width multiplier)
    sigma_t_func: callable returning extinction coefficient at a point
    """
    radiance = 0.0
    # 1. Query intersecting beams (simplified linear scan; replace with BVH traversal in practice)
    for beam in photon_beams:
        # 2. Compute cross-section at intersection
        # Assume intersection parameter t_c along camera ray is known from BVH query
        t_c = 0.0 
        intersection_pt = (camera_origin[0] + t_c * camera_dir[0],
                           camera_origin[1] + t_c * camera_dir[1],
                           camera_origin[2] + t_c * camera_dir[2])
        
        # Project intersection point onto beam direction to find t_b
        dx = intersection_pt[0] - beam['origin'][0]
        dy = intersection_pt[1] - beam['origin'][1]
        dz = intersection_pt[2] - beam['origin'][2]
        t_b = (dx * beam['dir'][0] + dy * beam['dir'][1] + dz * beam['dir'][2])
        
        # Get differential ray positions at t_b
        r_center = (beam['origin'][0] + t_b * beam['dir'][0],
                    beam['origin'][1] + t_b * beam['dir'][1],
                    beam['origin'][2] + t_b * beam['dir'][2])
        r_u = (beam['origin_u'][0] + t_b * beam['dir_u'][0],
               beam['origin_u'][1] + t_b * beam['dir_u'][1],
               beam['origin_u'][2] + t_b * beam['dir_u'][2])
        r_v = (beam['origin_v'][0] + t_b * beam['dir_v'][0],
               beam['origin_v'][1] + t_b * beam['dir_v'][1],
               beam['origin_v'][2] + t_b * beam['dir_v'][2])
        
        # Semi-axes
        a = math.sqrt(sum((r_center[i] - r_u[i])**2 for i in range(3)))
        b = math.sqrt(sum((r_center[i] - r_v[i])**2 for i in range(3)))
        
        # Effective radius for 2D measure, scaled by alpha
        r_eff = alpha * math.sqrt(a * b)
        area = math.pi * r_eff * r_eff
        
        # 3. Compute Transmittance
        # For 2D estimate, evaluate transmittance at the intersection point
        sigma_t = sigma_t_func(intersection_pt)
        dist = math.sqrt(sum((intersection_pt[i] - beam['origin'][i])**2 for i in range(3)))
        transmittance = math.exp(-sigma_t * dist)
        
        # 4. Add contribution
        # Phase function evaluation (dot product of directions)
        dot_prod = (camera_dir[0] * beam['dir'][0] + camera_dir[1] * beam['dir'][1] + camera_dir[2] * beam['dir'][2])
        phase = 0.25 * math.pi * (1.0 - dot_prod**2) # Standard isotropic approximation [G]
        radiance += (phase * beam['power'] * transmittance) / area
        
    return radiance
```

## Subtopics

### Light Source Emission with Differentials
This variant defines the initial geometry of photon beams based on the light source type to ensure tight packing and correct angular coverage [S1].
-   **Singularity Lights (Point/Spot):** Differential ray origins are set to the light source location [S1]. Directions are set to cover the angular extent with minimal overlap, where the solid angle per beam is $1/(pdf(\omega) \cdot N)$ [S1].
-   **Area Lights:** The surface is discretized into a jittered grid [S1]. Central ray origins are at grid centers, differential origins at grid edges [S1]. Directions follow the angular emission PDF [S1].

### Multiple Scattering Beam Width
For multiple scattering events, beam widths are set to cylinders with widths equal to the footprint of the parent beam at the scattering location $\mathbf{x}$ [S1]. The differential rays are always parallel to the parent beam [S1]. Unlike pdf-based or pilot density estimate methods which produced visibly biased results due to extreme spreading, this automatic cylinder method produces reasonable results [S1].

### Oblique Incident Light Angles
Compute locations along the refracted incident beam during numerical integration [S2, unofficial]. The explicit integration formulation allows choosing any $\vec{\omega}$ not orthogonal to the surface normal without modifications [S2, unofficial]. Similar to Yan et al. [YZXW12] who formulated QD for oblique angles by offsetting Gaussian centers, this method uses explicit integration to handle angles [S2, unofficial].

### Finite Slab Handling via Mirroring
Treat each sample (summand of Equation 8) as a dipole and mirror it at the extrapolated boundary to perform a multipole extension for a finite slab, similar to Donner and Jensen [DJ05] [S2, unofficial]. When close to $\mathbf{x}=0$, placing dipoles directly on reflective or transmissive boundaries can cause issues [S2, unofficial]. The equiangular sampling in the MIS strategy avoids placing dipoles directly on these boundaries, allowing for very thin slabs without drawbacks [S2, unofficial].

## Operating characteristics
-   **Time Complexity:** Dominated by the BVH traversal for beam intersection [S1].
-   **Memory Complexity:** Linear in the number of photon beams [S1]. Heterogeneous media require storing 1D transmittance functions, which can be significantly compressed for efficient storage [S1].
-   **Performance:** Converges significantly faster than photon points for sharp features [S1]. In a volume caustic scene, 32k beams converged in 43 seconds, while 512k photons took nearly twice as long and produced a noisier, over-blurred result [S1].
-   **Strengths:** Eliminates shadow rays by reusing visibility from photon tracing; handles area lights and multiple bounces naturally; avoids singularities via finite cross-sections [S1].
-   **Limitations:** Adaptive width based on differentials works well for single scattering and caustics but is less successful for multiple scattering [S1]. Fixed-width beams cause banding [S1].

## Background
Adaptive Beam Width is part of the photon beam framework introduced by Jarosz et al. [2008] as a generalization of photon mapping [S1]. It builds on the concept of photon differentials from Igehy [1999] and surface-based methods [Schjøth et al. 2007; Fabianowski and Dingliana 2009] [S1]. The method was developed to provide a compact, expressive intermediate representation for lighting in participating media, allowing for efficient radiance estimation without shadow rays [S1].

## Sources
[S1] A comprehensive theory of volumetric radiance estimation using photon points and beams, https://doi.org/10.1145/1899404.1899409, fetched 2026-10-26
[S2] Photon Beam Diffusion for Rendering, https://www.scribd.com/document/255860491/Photon-Beam-Diffusion, fetched 2026-10-29 (unofficial)
