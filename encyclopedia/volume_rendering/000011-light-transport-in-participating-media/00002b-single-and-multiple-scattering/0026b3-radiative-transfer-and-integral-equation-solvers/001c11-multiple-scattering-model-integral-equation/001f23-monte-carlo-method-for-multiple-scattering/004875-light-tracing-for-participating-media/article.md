# Light Tracing for Participating Media

Also called: Monte Carlo light tracing with random distance, Pattanaik and Mudur, Pattanaik and Mudur light tracing, Pattanaik et al. method, Random distance sampling (Light Tracing)

## Overview
Light tracing for participating media is a Monte Carlo method that solves the radiative transfer equation by tracing bundles of light from the source through the volume to find interaction points [S1][S2, book]. It differs from eye-tracing (path tracing) by generating paths from the light source, making it particularly effective for scenes with complex lighting or when the camera view is not the primary constraint [S2, book]. Unlike constant distance sampling methods, this approach is unbiased because it samples the distance to the next interaction continuously rather than restricting it to multiples of a discrete step size [S2, book].

## How it works
### Inputs and Outputs
**Inputs:**
*   Volume geometry and bounds.
*   Extinction coefficient field $\sigma_t(\mathbf{x})$ (or optical thickness $\kappa_t$).
*   Scattering coefficient $\sigma_s(\mathbf{x})$ and absorption coefficient $\sigma_a(\mathbf{x})$, where $\sigma_t = \sigma_s + \sigma_a$.
*   Phase function $p(\omega_i, \omega_o)$.
*   Light source emission distribution.
*   Rendering algorithm parameters (number of bundles, variance reduction flags).

**Outputs:**
*   Radiance values at image plane pixels or voxel storage buffers.

### Data Structures
The method typically uses a storage scheme suited for isotropic scattering. The specific layout depends on the implementation (image-based vs. voxel-based), but generally involves:
*   **Bundle State:** Current position $\mathbf{x}$, direction $\omega$, current flux/weight $W$, and accumulated optical depth.
*   **Storage Buffer:** If using voxel storage, a buffer mapping spatial coordinates to accumulated radiance. For image-based light tracing, a pixel buffer accumulates contributions when bundles hit the image plane [S1][S2, book].

### Numbered Steps

1.  **Initialize Bundle:** Shoot a light bundle from the light source with initial direction $\omega_0$ and weight $W_0 = 1.0$ (or scaled by source intensity).

2.  **Sample Distance to Interaction:**
    Sample the distance $s$ to the next interaction point (absorption or scattering) using the cumulative distribution function (CDF) derived from the extinction coefficient [S1][S2, book]:
    $$ F(s) = 1 - e^{-\sigma_t s} $$
    To sample $s$ given a uniform random number $\xi \in [0, 1)$:
    $$ s = -\frac{\ln(1 - \xi)}{\sigma_t} $$
    *Note: If $\sigma_t$ varies spatially, the optical depth $\tau$ is integrated along the ray, and the CDF is $1 - \exp(-\tau)$ [S1]. The distance $S$ is the total distance traveled [S1].*

3.  **Move Bundle:**
    Update the bundle position: $\mathbf{x}_{new} = \mathbf{x}_{old} + s \cdot \omega$.
    Check for boundary intersection. If the bundle exits the volume, terminate the path.

4.  **Determine Interaction Type:**
    At $\mathbf{x}_{new}$, decide if the event is absorption or scattering. Two methods are described in the sources:

    *   **Method A: Simple Absorption**
        Perform a sampling process to decide the event type based on the ratio of scattering to extinction $\Omega = \frac{\sigma_s}{\sigma_t}$ [S1][S2, book].
        *   Generate $\xi \in [0, 1)$.
        *   If $\xi < \Omega$: **Scattering**. Update weight $W = W \cdot \frac{\sigma_s}{\sigma_t}$ (or similar Russian Roulette/weight adjustment depending on implementation details not fully specified, but typically the weight is multiplied by the albedo $a = \sigma_s/\sigma_t$ if using absorption suppression, or handled via Russian Roulette in standard MC). *Correction based on standard MC and source context:* In Simple Absorption, if scattering occurs, the weight is usually multiplied by the albedo $a = \sigma_s/\sigma_t$ to account for the probability of scattering vs absorption, or the path continues with weight $W \cdot a$. If absorption occurs, the path terminates.
        *   If $\xi \ge \Omega$: **Absorption**. Terminate the path.

    *   **Method B: Absorption Suppression**
        The bundle **always** scatters. The flux is reduced by multiplying it by the ratio $\Omega = \frac{\sigma_s}{\sigma_t}$ [S1][S2, book].
        $$ W_{new} = W_{old} \cdot \frac{\sigma_s}{\sigma_t} $$
        This avoids path termination due to absorption but reduces the weight, increasing variance.

5.  **Scattering Event:**
    If scattering occurs (either via Simple Absorption selection or Absorption Suppression):
    *   Sample the new direction $\omega_{new}$ from the phase function $p(\omega_{old}, \omega_{new})$.
    *   Update direction: $\omega = \omega_{new}$.
    *   Apply variance reduction techniques if enabled (see below).
    *   Repeat from Step 2.

6.  **Termination:**
    *   If the bundle hits the image plane, add $W$ to the corresponding pixel [S1][S2, book].
    *   If the bundle exits the volume, terminate.
    *   If absorption occurs in Simple Absorption, terminate.

### Variance Reduction Techniques
The sources propose three specific techniques [S1][S2, book]:
1.  **Forced Interaction:** Force a bundle to interact with each voxel it passes through. This ensures uniform sampling density across the volume but may introduce bias if not handled carefully with weight adjustments.
2.  **Absorption Suppression:** As described in Step 4, always scatter and reduce weight. This reduces variance from absorption events but increases the number of scattering steps.
3.  **Particle Divergence:** At scattering points, split the outgoing bundle into many smaller bundles. This increases the number of rays traveling through the volume, improving the sampling of energy leaving the volume, especially in sparse or complex media [S1][S2, book].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def light_trace_participating_media(light_source, volume, camera, num_bundles):
    """
    Monte Carlo light tracing for participating media.
    Uses random distance sampling (Pattanaik et al. method).
    """
    pixel_buffer = zeros_like(camera.image_shape)
    
    for _ in range(num_bundles):
        # 1. Initialize bundle from light source
        origin, direction, weight = light_source.sample()
        
        current_pos = origin
        current_dir = direction
        current_weight = weight
        
        while current_weight > 0.001: # Threshold for termination
            # 2. Sample distance to next interaction
            # CDF: F(s) = 1 - exp(-sigma_t * s)
            # s = -ln(1 - xi) / sigma_t
            xi = random_uniform()
            sigma_t = volume.get_extinction(current_pos)
            
            # Handle zero extinction (vacuum)
            if sigma_t == 0:
                # Move to next boundary or far point
                next_boundary_dist = volume.get_next_boundary(current_pos, current_dir)
                if next_boundary_dist is None:
                    break
                current_pos += current_dir * next_boundary_dist
                continue
            
            s = -log(1.0 - xi) / sigma_t
            
            # 3. Move bundle
            next_pos = current_pos + current_dir * s
            
            # Check if interaction is within volume bounds
            if not volume.contains(next_pos):
                # Hit boundary, terminate or reflect/refract depending on model
                break
            
            # 4. Determine interaction type
            sigma_s = volume.get_scattering(current_pos)
            sigma_a = volume.get_absorption(current_pos)
            sigma_t = sigma_s + sigma_a
            
            if sigma_t == 0:
                current_pos = next_pos
                continue
                
            albedo = sigma_s / sigma_t
            
            # Using Absorption Suppression method as described in sources
            # Always scatter, reduce weight by albedo
            current_weight *= albedo
            
            if current_weight < 1e-6:
                break
                
            # 5. Scattering
            # Sample new direction from phase function
            new_dir = volume.phase_function.sample(current_dir)
            
            # Update state
            current_pos = next_pos
            current_dir = new_dir
            
            # 6. Check if hits image plane
            # Simplified: if ray intersects image plane, add contribution
            hit, hit_point, hit_pixel = camera.ray_intersect(current_pos, current_dir)
            if hit:
                pixel_buffer[hit_pixel] += current_weight
                
            # Optional: Variance Reduction - Particle Divergence
            # If enabled, split bundle into multiple rays here
            # For simplicity, we continue with single ray in this basic implementation
            
    return pixel_buffer
```

### Edge Cases and Pitfalls
*   **Anisotropic Scattering:** The basic storage scheme is suited for isotropic scattering. For anisotropic scattering, the storage scheme must be modified to account for directionality [S1][S2, book].
*   **Precision:** Sampling distance continuously avoids the bias of constant distance sampling. Using constant distance sampling leads to biased results because bundles can only be scattered at distances which are multiples of a step size $\Delta$, and the expected length before scattering won't be equal to the mean free path without absorption [S2, book].
*   **Performance:** To get accurate sampling of energy leaving the volume, the number of rays must be higher than in per-voxel storage methods [S1][S2, book].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of bundles and $M$ is the average number of scattering events per bundle.
*   **Memory Complexity:** $O(P)$ for pixel buffer or $O(V)$ for voxel storage, where $P$ is pixel count and $V$ is voxel count.
*   **Strengths:** Unbiased result (unlike constant distance sampling); effective for complex lighting; no need for eye-point sampling [S2, book].
*   **Limitations:** Requires many bundles for accurate sampling of energy leaving the volume; variance can be high in sparse media without variance reduction techniques [S1][S2, book].
*   **When to use:** Scenes with complex light sources or when light tracing is more efficient than path tracing (e.g., caustics in participating media).
*   **When not to use:** Simple homogeneous media where analytical solutions or simpler methods suffice; scenes where eye-tracing is more efficient (e.g., small light sources).

## Background
The method was introduced by Pattanaik and Mudur [66] and Pattanaik et al. [14] as a Monte Carlo light tracing technique for participating media. It uses a sampling process to find points of interaction within the volume [S1][S2, book]. The method is part of the broader family of Monte Carlo methods for solving the radiative transfer equation, specifically addressing multiple scattering [S2, book]. It is often compared to bidirectional path tracing for non-emitting participating media, which combines light shooting and light gathering [S2, book].

## Sources
[S1] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
[S2] Global Illumination Techniques for the Simulation of Participating Media, https://inria.hal.science/inria-00510096v1, fetched 2026-10-26 (book)
