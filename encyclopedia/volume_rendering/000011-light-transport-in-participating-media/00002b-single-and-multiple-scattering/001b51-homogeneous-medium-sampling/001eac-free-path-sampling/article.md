# Free-path sampling
Also called: Homogeneous Volume Tracking, Single Scattering Volume Path Tracing with Transmittance Sampling, Single Scattering with Surface Emission, Tracking in Homogeneous Volumes, Vol Path Tracing 1

## Overview
Free-path sampling determines the distance a ray travels in a homogeneous participating medium before interacting, avoiding explicit distance and spherical integrations by letting the ray randomly bounce within the medium before potentially hitting a surface [S1, unofficial].

## How it works
**Inputs:** Ray origin $\mathbf{x}$, direction $\boldsymbol{\omega}$, maximum travel distance $t_{\max}$, medium extinction coefficient $\sigma_t = \sigma_a + \sigma_s$, and scene geometry providing the nearest surface intersection distance $d_{\text{surf}}$ along $\boldsymbol{\omega}$ [S1, unofficial].
**Outputs:** Interaction distance $t$, interaction type (volume or surface), and radiance contribution [S1, unofficial].

**Data Structures:**
*   `Ray`: `origin` (vec3), `direction` (vec3), `t_max` (float).
*   `Medium`: `sigma_a` (float), `sigma_s` (float), `sigma_t` (float).
*   `Intersection`: `point` (vec3), `distance` (float), `valid` (bool).

**Steps:**
1.  **Sample Free Flight Distance:** Generate a uniform random number $\xi \in [0, 1)$ and calculate the free flight distance $t_{\text{free}}$ using the inverse cumulative distribution function of the exponential distribution: $t_{\text{free}} = -\ln(\xi) / \sigma_t$ [S1, unofficial].
2.  **Determine Interaction Type:** Compare $t_{\text{free}}$ with the distance to the nearest surface intersection $d_{\text{surf}}$ [S1, unofficial].
    *   If $t_{\text{free}} < d_{\text{surf}}$: The ray interacts with the volume [S1, unofficial].
    *   If $t_{\text{free}} \ge d_{\text{surf}}$: The ray hits the surface before interacting with the volume [S1, unofficial].
3.  **Handle Volume Interaction:** Compute the transmittance from the ray origin to the interaction point $\mathbf{x}_t = \mathbf{x} + t_{\text{free}}\boldsymbol{\omega}$ as $T(\mathbf{x}, \mathbf{x}_t) = e^{-\sigma_t t_{\text{free}}}$ [S1, unofficial]. Sample a new direction $\boldsymbol{\omega}'$ from the phase function $p(\boldsymbol{\omega}' \leftarrow \boldsymbol{\omega})$ at $\mathbf{x}_t$ [S1, unofficial]. Compute the in-scattered radiance contribution and handle Next Event Estimation (NEE) for each light, plus indirect lighting via the phase function [S1, unofficial].
4.  **Handle Surface Interaction:** Execute standard path tracer code at $d_{\text{surf}}$, including NEE, BRDF evaluation, and MIS [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import random
import math

def free_path_sample(origin, direction, sigma_t, sigma_s, sigma_a, phase_func, scene_intersect, lights):
    # Step 1: Sample free flight distance
    xi = random.random()
    if sigma_t == 0.0:
        t_free = float('inf')
    else:
        t_free = -math.log(xi) / sigma_t
        
    # Step 2: Find nearest surface intersection
    hit = scene_intersect(origin, direction)
    d_surf = hit.distance if hit.valid else float('inf')
    
    # Step 3: Determine interaction type
    if t_free < d_surf:
        # Volume interaction
        t = t_free
        x_t = [origin[i] + t * direction[i] for i in range(3)]
        transmittance = math.exp(-sigma_t * t)
        
        # Sample phase function for new direction [G]
        omega_new = phase_func(direction)
        
        # Compute in-scattered radiance (NEE for each light) [S1, unofficial]
        radiance = 0.0
        for light in lights:
            # Standard NEE geometry term and PDF evaluation [G]
            radiance += light.sample_and_eval(x_t, omega_new)
            
        # Recursive call for indirect lighting
        radiance += free_path_sample(x_t, omega_new, sigma_t, sigma_s, sigma_a, phase_func, scene_intersect, lights)
        
        return radiance * transmittance
    else:
        # Surface interaction
        # Standard path tracing logic for BRDF/NEE/MIS [G]
        return surface_path_trace(hit)

def scene_intersect(origin, direction):
    # Standard ray-scene intersection routine [G]
    return type('Hit', (), {'valid': False, 'distance': float('inf')})()

def surface_path_trace(hit):
    # Standard path tracing logic for BRDF/NEE/MIS [G]
    return 0.0
```

**Parameters:**
*   $\sigma_t$: Extinction coefficient. Not given in the sources.
*   $\xi$: Uniform random number.
*   Anisotropy: Parameter for the Henyey-Greenstein phase function ranging from $-1$ to $1$ [S1, unofficial].

**Edge Cases:**
*   If $\sigma_t = 0$, the medium is transparent; set $t_{\text{free}} = \infty$ to ensure surface intersection is handled [G].
*   Precision issues may arise if $t_{\text{free}}$ is very close to $d_{\text{surf}}$; a small epsilon bias may be needed to prevent self-intersection artifacts [G].

## Subtopics

### Unidirectional transmittance-based path sampling
A method to construct paths by incrementally adding vertices from one endpoint until a light source or sensor is hit, sampling propagation distance proportional to transmittance [S2]. It differs from standard free-path sampling by explicitly building the path vertex-by-vertex with direction sampled proportionally to the phase function and distance sampled proportionally to transmittance, rather than tracking a single ray bounce [S2]. Choose this when constructing paths from a specific endpoint with transmittance-weighted sampling is required [S2].

## Operating characteristics
Time complexity is not given in the sources [S1, unofficial]. Memory complexity is not given in the sources [S1, unofficial]. The method was considered unbiased but produced noticeable noise; it was abandoned in the referenced implementation because it handles multiple scattering, which was not the project's focus [S1, unofficial]. Strengths include simple implementation for single scattering events [S1, unofficial]. Limitations include high noise in multiple scattering scenarios [S1, unofficial]. It should be used for single scattering dominated scenes [S1, unofficial]. It should not be used for scenes requiring efficient multiple scattering handling [S1, unofficial].

## Background
Free-path sampling originates from fundamental light transport principles in participating media, specifically the Beer-Lambert law governing transmittance [S1, unofficial]. It is a core component of volumetric path tracing algorithms for simulating light interacting with homogeneous volumes [S1, unofficial]. The method was discussed in the context of implementing volumetric effects in path tracers, where it was ultimately abandoned due to noise from multiple scattering handling [S1, unofficial].

## Sources
[S1] l0exy1h/VolumetricPathTracing-public, https://github.com/l0exy1h/VolumetricPathTracing-public, fetched 2026-10-26 (unofficial)
[S2] Joint importance sampling of low-order volumetric scattering, https://doi.org/10.1145/2508363.2508411, fetched 2026-10-28
