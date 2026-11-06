# Single Scattering (Blinn Model)
Also called: Blinn 1982 Cloud Model

## Overview
The Single Scattering (Blinn Model) is a light transport approximation for participating media that accounts for exactly one secondary scattering event between the observer and a light source [S1, unofficial]. It solves the problem of rendering volumetric effects like clouds or Saturn's rings by assuming low albedo, thereby ignoring multiple scattering interactions [S1, unofficial]. It differs from the general Single Scattering Model by specifically relying on Blinn's 1982 approximations for uniform, low-density media, and from the Single Scattering Propagator by using a direct ray-tracing integration approach rather than precomputed propagation fields [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A viewing ray originating from the camera, a light source position, a volume density field $\rho(\mathbf{x})$, and a phase function $p(\cos \theta)$.
*   **Output:** The radiance contribution $L$ at the pixel corresponding to the viewing ray.

**Data Structures:**
*   **Volume Density:** A scalar field $\rho(\mathbf{x})$ representing the scattering coefficient at position $\mathbf{x}$.
*   **Phase Function:** A function $p(\cos \theta)$ mapping the cosine of the scattering angle $\theta$ to a relative intensity weight.

**Parameters:**
*   **Albedo:** The proportion of light reflected by a particle relative to light absorbed. The model assumes low albedo [S1, unofficial].
*   **Phase Function Type:** Commonly Mie scattering for clouds, approximated as $p(\cos \theta) \approx w_0$ (perfectly diffuse) or $p(\cos \theta) = w_0 \frac{3}{4}(1 + \cos^2 \theta)$ (Rayleigh) [S2, unofficial].

**Steps:**
1.  **Ray Intersection:** Determine the entry point $\mathbf{x}_{in}$ and exit point $\mathbf{x}_{out}$ of the viewing ray within the volume bounding box.
2.  **Light Ray Intersection:** For each sample point $\mathbf{x}$ along the viewing ray, cast a secondary ray towards the light source.
3.  **Transmittance Calculation:** Compute the optical depth $\tau$ along the secondary ray from $\mathbf{x}$ to the light source (or until it exits the volume). The transmittance is $T = \exp(-\tau)$.
4.  **Scattering Integration:** Integrate the scattering contribution along the viewing ray. The brightness contribution $I$ for a light source is calculated as [S2, unofficial]:
    $$ I_i(x, y, z) = \exp\left(-\int_{0}^{r} \tau \rho(\xi) d\xi\right) \int_{0}^{r} \rho(\xi(t), \eta(t), \zeta(t)) p(\cos \theta) dt $$
    Where $\tau = r \rho$ is the optical depth, and the integral is performed along the path from the volume entry to the current point [S2, unofficial].
5.  **Phase Function Evaluation:** Evaluate $p(\cos \theta)$ where $\theta$ is the angle between the incoming light direction and the outgoing view direction [S1, unofficial].
6.  **Accumulation:** Sum the contributions from all sample points along the viewing ray.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def blinn_single_scattering(view_ray, light_source, volume, phase_func):
    # 1. Find intersection with volume bounding box
    t_min, t_max = view_ray.intersect_bbox(volume.bbox)
    
    # 2. Initialize accumulated radiance
    radiance = 0.0
    
    # 3. Sample along the viewing ray
    num_samples = 100
    step_size = (t_max - t_min) / num_samples
    
    for i in range(num_samples):
        t = t_min + i * step_size
        point = view_ray.origin + view_ray.direction * t
        
        # 4. Cast secondary ray to light source
        light_ray = Ray(point, light_source.position - point)
        
        # 5. Calculate transmittance from point to light
        # Simplified: assume light is outside or handle intersection
        t_light_min, t_light_max = light_ray.intersect_bbox(volume.bbox)
        
        # Optical depth along light ray
        optical_depth = 0.0
        num_light_samples = 50
        light_step = (t_light_max - t_light_min) / num_light_samples
        
        for j in range(num_light_samples):
            t_l = t_light_min + j * light_step
            light_point = light_ray.origin + light_ray.direction * t_l
            density = volume.density(light_point)
            optical_depth += density * light_step
            
        transmittance = exp(-optical_depth)
        
        # 6. Calculate scattering contribution
        # Direction from point to light
        light_dir = normalize(light_source.position - point)
        view_dir = -view_ray.direction # Direction towards camera
        
        # Angle theta between light_dir and view_dir
        cos_theta = dot(light_dir, view_dir)
        
        # Phase function value
        phase = phase_func(cos_theta)
        
        # Density at current point
        density = volume.density(point)
        
        # Contribution: density * transmittance * phase * step_size
        # Note: The formula in [S2, unofficial] combines precomputed brightness
        # with ray tracing. Here we compute the differential contribution.
        radiance += density * transmittance * phase * step_size
        
    return radiance
```

**Edge Cases and Pitfalls:**
*   **High Albedo:** The model fails for media with high albedo as multiple scattering becomes significant [S1, unofficial].
*   **Boundary Conditions:** The path boundaries for the light ray must be correctly clipped to the volume extent to avoid incorrect optical depth calculations [S2, unofficial].
*   **Precision:** Numerical integration errors can accumulate, especially with steep density gradients. Romberg integration is suggested for higher accuracy [S2, unofficial].

## Subtopics

### Kajiya-Von Herren Ray Tracing Algorithm
This variant uses a precomputed brightness contribution $I_i(x, y, z)$ for each light source, calculated via Romberg integration [S2, unofficial]. The path boundaries are defined as $\lambda_1 = \max(0, d_1)$ and $\lambda_2 = \min(d_{global}, d_2)$, where $d_1$ is the distance to the nearest intersection with the bounding extent, $d_{global}$ is the distance to the nearest intersection with the rest of the world database, and $d_2$ is the distance to the farthest intersection with the bounding extent [S2, unofficial]. It is chosen when precomputation is feasible to accelerate per-frame rendering [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$ where $N$ is the number of samples along the viewing ray and $M$ is the number of samples along the light ray.
*   **Memory Complexity:** $O(V)$ where $V$ is the size of the volume density field.
*   **Strengths:** Simple to implement; efficient for low-density media like clouds or Saturn's rings [S1, unofficial].
*   **Limitations:** Inaccurate for high-albedo or thick media; ignores multiple scattering [S1, unofficial].
*   **When to Use:** Real-time or interactive rendering of thin volumetric effects [S1, unofficial].
*   **When Not to Use:** Dense fog, smoke, or scenes requiring realistic global illumination within the volume [S1, unofficial].

## Background
The model was published by James F. Blinn in 1982 for cloud simulation [S1, unofficial]. It was one of the first works to incorporate reflection and scattering in participating media, specifically targeting the rings of Saturn [S2, unofficial]. It assumes a uniform medium of relatively low albedo [S2, unofficial]. Later, Max extended this model to arbitrary shapes [S1, unofficial], and Bouthor added anisotropic dispersion effects [S1, unofficial].

## Sources
[S1] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
[S2] Ray Tracing for Volume Densities, https://www.scribd.com/document/746685923/ray-tracking-volume-densities, fetched 2026-10-25 (unofficial)
