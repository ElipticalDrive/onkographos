# Monte Carlo Method for Multiple Scattering

Also called: Monte Carlo Volume Rendering, Stochastic Ray Tracing for Participating Media

## Overview
The Monte Carlo method solves the integral equation for directional scattering intensity $I(X, \omega)$ by stochastically tracing sample rays from the eye through a pixel [S1, unofficial]. It differs from deterministic solvers (like the P-N method or Diffusion Approximation) by relying on random absorption and scattering events based on medium properties, making it unbiased but inherently noisy due to the low probability of eye-to-light ray connections [S1, unofficial].

## How it works
**Inputs:**
*   Camera position and pixel sampling pattern.
*   Volume medium properties: extinction coefficient $\tau$, albedo $a$, and phase function $p$.
*   Light sources (emitters).

**Outputs:**
*   Pixel intensity (flux contribution).

**Data Structures:**
*   Ray state: position $X$, direction $\omega$, current weight.
*   Medium lookup: voxel grid or texture map storing $\tau, a, p$ at location $X$.
*   Caustic Texture Map (optional, for hybrid approaches): stores scattered radiance $I(X_i, \omega)$ or isotropic scattering contributions from light sources.

**Steps:**
1.  **Ray Generation:** For each pixel, generate a primary ray from the eye into the scene.
2.  **Interaction Sampling:** As the ray traverses the volume, determine interaction points. At each point, decide between absorption or scattering based on the albedo $a$ and extinction coefficient $\tau$ [S1, unofficial].
3.  **Direction Selection:** If scattering occurs, select the new direction $\omega'$ using the phase function $p$ [S1, unofficial].
4.  **Contribution Check:** If the ray reaches a light source or volume emitter, accumulate its flux to the pixel intensity [S1, unofficial].
5.  **Termination:** If the ray is absorbed or leaves the volume without hitting a source, terminate the path.

**Hybrid Approach (Caustic Textures):**
To mitigate noise, a two-pass hybrid method can be used [S1, unofficial]:
1.  **Light Pass:** Propagate light from sources using Monte Carlo. Store scattering contributions at voxels $X_i$ into a texture map. This can store isotropic scattering or generalized directional scattered texture $I(X_i, \omega)$ [S1, unofficial].
2.  **Eye Pass:** Trace rays from the eye. Instead of full path tracing, use the pre-computed texture map to account for interreflections, performing a final Monte Carlo bounce toward the viewpoint [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def monte_carlo_volume_rendering(camera, volume, lights):
    pixel_intensity = 0.0
    # Generate ray from eye through pixel
    ray = camera.generate_ray()
    current_weight = 1.0
    
    while current_weight > epsilon:
        # Find intersection with volume boundary or next voxel
        hit = volume.ray_intersect(ray)
        if hit is None:
            break
            
        # Sample distance to next interaction based on extinction coefficient tau
        # Distance d is sampled from exponential distribution with mean 1/tau
        d = sample_distance(hit.distance, volume.extinction_coefficient(hit.position))
        
        if hit.distance < d:
            # Ray exits volume or hits object before interaction
            ray = ray.advance(hit.distance)
            continue
            
        # Interaction occurs at hit.position + d
        interaction_pos = ray.position + ray.direction * d
        
        # Determine if scattering or absorption based on albedo a
        if random() < volume.albedo(interaction_pos):
            # Scattering event
            # Sample new direction omega' from phase function p
            new_direction = sample_phase_function(volume.phase_function, ray.direction)
            
            # Update ray
            ray = Ray(interaction_pos, new_direction)
            
            # Update weight by albedo and phase function normalization
            current_weight *= volume.albedo(interaction_pos) * volume.phase_function(ray.direction, new_direction)
        else:
            # Absorption event
            break
            
        # Check if ray hits a light source
        if ray.hits_light(lights):
            pixel_intensity += current_weight * light_contribution(ray, lights)
            
    return pixel_intensity
```

**Parameters:**
*   $\tau$ (extinction coefficient): Controls mean free path.
*   $a$ (albedo): Probability of scattering vs. absorption.
*   $p$ (phase function): Angular distribution of scattered light.
*   Number of samples: High values required to reduce variance [S1, unofficial].

**Edge Cases:**
*   **Low Probability Paths:** Contributing rays are a small fraction of all considered rays, leading to high variance [S1, unofficial].
*   **Boundary Conditions:** Rays may exit the volume before interacting; proper handling of exit distances is required.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot S)$, where $N$ is the number of pixels and $S$ is the number of samples per pixel. Convergence is slow ($1/\sqrt{S}$) [S1, unofficial].
*   **Memory Complexity:** $O(V)$ for volume storage, plus $O(C)$ for caustic textures if used [S1, unofficial].
*   **Performance:** Images tend to appear noisy and computation takes a long time because many samples are required to decrease the variance of the mean of contributions [S1, unofficial].
*   **Strengths:** Unbiased solution; handles complex geometries and anisotropic scattering naturally.
*   **Limitations:** High noise levels; inefficient for scenes where eye-to-light paths are rare.

## Background
The Monte Carlo method was originally developed by physicists for neutron transport and applied to rendering surface interreflection by Cook et al. and Kajiya [S1, unofficial]. It was applied to volume applications by Rushmeier [S1, unofficial]. The hybrid approach using caustic texture maps was proposed by Shirley, Chen et al., and Heckbert, and adapted for volume rendering by Blasi et al. [S1, unofficial].

## Sources
[S1] Optical Models for Direct Volume Rendering - Duke Computer Science, https://courses.cs.duke.edu/spring03/cps296.8/papers/max95opticalModelsForDirectVolumeRendering.pdf, fetched 2026-10-26 (unofficial)
