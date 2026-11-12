# Constant Distance Sampling

Also called: Blasi et al. [4, 5] method, Blasi et al, Blasi et al. constant distance sampling, Monte Carlo light tracing with constant step

## Overview
Constant Distance Sampling is a Monte Carlo light tracing technique for rendering participating media that advances light bundles in discrete steps of fixed length $\delta$ [S1][S2, book]. It solves the problem of simulating multiple scattering in homogeneous media by approximating the continuous transport equation with a grid-aligned discrete process [S1][S2, book]. Unlike [Free-path sampling](1844-free-path-sampling.md), which samples interaction distances from the exact transmittance distribution, Constant Distance Sampling restricts scattering events to multiples of $\delta$, introducing a deterministic bias in exchange for simplified storage and traversal logic [S2, book]. It is distinct from [Majorant Iterator Ray Sampling](3075-majorant-iterator-ray-sampling.md) in its use of a fixed step rather than a majorant-based rejection loop, and from [Heterogeneous Medium Sampling](6654-heterogeneous-medium-sampling.md) as it assumes constant coefficients within the medium [S1][S2, book].

## How it works

### Inputs and Outputs
**Inputs:**
*   Scene geometry (surfaces classified as diffuse or specular).
*   Homogeneous medium properties: absorption coefficient $\sigma_a$, scattering coefficient $\sigma_s$, and extinction coefficient $\sigma_t = \sigma_a + \sigma_s$.
*   Step size $\delta$.
*   Phase function approximation (Schlick phase function).
*   Light sources (for light tracing).

**Outputs:**
*   Radiance contribution stored in voxels (isotropic component) and surfaces (diffuse/specular).
*   Final image or progressive refinement sequence.

### Data Structures
*   **Voxel Grid:** Stores the isotropic part of the scattered energy. The directional component is not stored in the voxel for subsequent illumination [S1][S2, book].
*   **Surface Energy Buffer:** Stores unshot energy for diffuse surfaces in mixed scenes [S2, book].

### Parameters
*   $\delta$: Constant step size. Smaller values reduce bias but increase computation time [S2, book].
*   Schlick phase function parameters: Approximates Mie scattering as a combination of isotropic and forward scattering components [S1][S2, book].

### Algorithm Steps
1.  **Initialization:** Create light bundles from light sources with initial flux $E$.
2.  **Propagation:** Advance the bundle along its direction by distance $\delta$.
3.  **Absorption:** Update the bundle's flux $E$ by multiplying it with the transmittance over the step $\delta$:
    $$ E_{new} = E_{old} \cdot e^{-\sigma_a \delta} $$
    This accounts for absorption along the entire path of the bundle [S1][S2, book].
4.  **Scattering Decision:** At the new position, perform a sampling process to decide if scattering occurs. The sources describe a "sampling process" but do not provide the explicit probability formula for the decision at each step. However, they note that bundles can only be scattered at distances which are multiples of $\delta$ [S2, book]. [G] In standard implementations of this bias, the probability of scattering at a step is often derived from the transmittance over $\delta$, i.e., $P_{scatter} = 1 - e^{-\sigma_t \delta}$, but the sources do not explicitly define this probability, only the consequence of the fixed step.
5.  **Scattering Event:** If scattering occurs:
    *   Determine the scatter direction using optimal importance sampling of the scattering component based on the Schlick phase function [S1][S2, book].
    *   Store the isotropic part of the scattered energy in the current voxel [S1][S2, book].
    *   Continue the bundle with the new direction and reduced flux.
6.  **Surface Interaction (Mixed Scenes):**
    *   If the bundle hits a surface:
        *   **Diffuse Surface:** Store the bundle's energy in the surface buffer and terminate the bundle's path [S2, book].
        *   **Specular Surface:** Reflect the bundle using importance sampling and continue [S2, book].
7.  **Exit:** If the bundle exits the medium, record its energy at the border voxel [S2, book].
8.  **Progressive Refinement (Optional):** In mixed scenes, if a bundle hits a diffuse surface, its path ends. At each iteration, bundles representing the unshot energy of the element with the highest value are spread to the environment [S2, book].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def constant_distance_sampling(light_bundles, medium, step_size, surfaces):
    """
    Simulates light transport in a homogeneous participating medium using constant distance sampling.
    
    Args:
        light_bundles: List of initial light bundles (position, direction, flux).
        medium: Object containing sigma_a, sigma_s, sigma_t.
        step_size: Constant step length delta.
        surfaces: List of surface objects with hit testing and reflection properties.
        
    Returns:
        voxel_energy: Accumulated isotropic energy in voxels.
        surface_energy: Accumulated energy on diffuse surfaces.
    """
    voxel_energy = initialize_voxel_grid()
    surface_energy = initialize_surface_buffer()
    
    active_bundles = light_bundles
    
    while active_bundles:
        next_active_bundles = []
        
        for bundle in active_bundles:
            # Step 1: Propagate and Absorb
            new_position = bundle.position + bundle.direction * step_size
            transmittance = exp(-medium.sigma_a * step_size)
            bundle.flux *= transmittance
            
            # Step 2: Check for Surface Intersection
            hit_surface, hit_point, surface_normal, surface_type = ray_intersect(new_position, bundle.direction, surfaces)
            
            if hit_surface:
                if surface_type == "diffuse":
                    # Store energy and terminate
                    surface_energy.add(hit_point, bundle.flux)
                    continue
                elif surface_type == "specular":
                    # Reflect and continue
                    bundle.direction = reflect(bundle.direction, surface_normal)
                    bundle.position = hit_point
                    next_active_bundles.append(bundle)
                    continue
            
            # Step 3: Scattering Decision
            # Note: Sources do not specify the exact probability formula for scattering
            # at each step, only that scattering is restricted to multiples of delta.
            # [G] Standard implementation uses P_scatter = 1 - exp(-sigma_t * delta)
            scatter_prob = 1.0 - exp(-medium.sigma_t * step_size)
            
            if random_uniform() < scatter_prob:
                # Scattering Event
                # Determine scatter direction using Schlick phase function importance sampling
                scatter_direction = sample_schlick_phase(bundle.direction, medium)
                
                # Store isotropic part in voxel
                # Note: Sources state only isotropic part is stored
                voxel_energy.add(bundle.position, bundle.flux * isotropic_component)
                
                # Continue bundle with new direction
                bundle.position = new_position
                bundle.direction = scatter_direction
                next_active_bundles.append(bundle)
            else:
                # No scattering, continue propagation
                bundle.position = new_position
                next_active_bundles.append(bundle)
                
        active_bundles = next_active_bundles
        
    return voxel_energy, surface_energy
```

### Edge Cases and Pitfalls
*   **Bias:** The method produces biased results because the sampling strategy restricts scattering to discrete distances (multiples of step size), causing the expected length before scattering to differ from the mean free path without absorption [S2, book].
*   **Variance vs. Bias Trade-off:** To assure results with a variance below a threshold, the time required is approximately of the order of the inverse of the step size $\delta$ [S2, book]. Smaller $\delta$ reduces bias but increases computation time [S2, book].
*   **Isotropic Energy Loss:** The isotropic part of the voxel is not considered for the illumination of other voxels. Since the directional component is much more important, the resulting error is expected not to be significant, but a progressive refinement strategy could be used when isotropic energy becomes too important [S1][S2, book].
*   **Mixed Scenes:** In mixed scenes, the storage scheme requires a higher number of rays traveling through the volumes to get an accurate sampling of the energy leaving the volume compared to per-voxel storage [S2, book].

## Operating characteristics

### Time and Memory Complexity
*   **Time Complexity:** The time required to assure results with a variance below a threshold is approximately of the order of the inverse of the step size $\delta$ [S2, book].
*   **Memory Complexity:** Depends on the voxel grid size for storing isotropic energy and surface buffers for diffuse surfaces [S1][S2, book].

### Measured Performance
*   Compared to random distance sampling [14], constant distance sampling [5] is biased and requires more computation time for the same variance threshold [S2, book].
*   For applications where the isotropic assumption can be used, hierarchical radiosity methods have better performance in computation time and are more reliable than progressive refinement methods like Blasi et al. [5] [S2, book].

### Strengths and Limitations
*   **Strengths:** Importance sampling using the Schlick phase function is quite inexpensive [S1][S2, book]. Can render mixed scenes with progressive refinement [S2, book].
*   **Limitations:** Biased results due to discrete scattering distances [S2, book]. Requires more computation time than random distance sampling for the same variance [S2, book]. Isotropic energy loss may affect accuracy in highly scattering media [S1][S2, book].

### When to Use
*   Use when computational efficiency of phase function sampling is prioritized over unbiased accuracy.
*   Use for homogeneous media where the bias introduced by $\delta$ is acceptable.
*   Use for mixed scenes requiring progressive refinement [S2, book].

### When Not to Use
*   Use when unbiased results are required (use [Free-path sampling](1844-free-path-sampling.md) instead) [S2, book].
*   Use when high accuracy in isotropic scattering is needed without progressive refinement [S1][S2, book].

## Background

### Origin
Constant Distance Sampling was described by Blasi et al. in [4, 5] (also cited as [9, 10] in [S1]). The first method [4] deals with a single participating medium, while the second [5] can render mixed scenes [S1][S2, book].

### History and Development
The method is part of the Monte Carlo light tracing family for participating media [S1][S2, book]. It uses an approximation to Mie scattering as a combination of isotropic plus forward scattering components, sampled via the Schlick phase function [S1][S2, book]. A progressive refinement technique was introduced in [5] to handle mixed scenes by classifying surfaces as diffuse or specular and accumulating unshot energy on diffuse surfaces [S2, book].

### Where it is used
Used in global illumination techniques for the simulation of participating media [S2, book]. Specifically in Monte Carlo light tracing for homogeneous media and mixed scenes [S1][S2, book].

## Sources

[S1] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
[S2] Global Illumination Techniques for the Simulation of Participating Media, https://inria.hal.science/inria-00510096v1, fetched 2026-10-26 (book)
