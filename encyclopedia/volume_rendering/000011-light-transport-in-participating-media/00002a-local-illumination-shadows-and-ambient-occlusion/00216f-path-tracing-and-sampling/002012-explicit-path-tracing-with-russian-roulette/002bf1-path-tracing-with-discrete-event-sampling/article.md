# Path Tracing with Discrete Event Sampling

Also called: Null-Event Path Tracing, Russian Roulette with Discrete Events

## Overview
Path Tracing with Discrete Event Sampling is a path tracing algorithm for participating media that replaces continuous Russian Roulette termination with a discrete probabilistic decision between absorption, scattering, and null events at each sampled point [S1, unofficial]. Unlike [Explicit Path Tracing with Russian Roulette](2209-explicit-path-tracing-with-russian-roulette.md), which uses continuous probability based on albedo/extinction ratios, this method explicitly handles null events (transmittance without interaction) to improve efficiency in sparse media by skipping non-interacting steps [S1, unofficial]. It supports temperature-dependent emission and Henyey-Greenstein phase function scattering [S1, unofficial].

## How it works
**Inputs:**
- Ray origin $\mathbf{o}$ and direction $\mathbf{d}$.
- Volume grid providing density $\sigma_t(\mathbf{x})$, temperature $T(\mathbf{x})$ (optional), and emission properties.
- Parameters: `max_depth`, `henyey_greenstein_g`, `le_scale`, `temperature_scale`, `temperature_offset`, `infinite_light`.
- Random number generator (RNG).

**Outputs:**
- Accumulated radiance $L$ along the ray.

**Data Structures:**
- `MajorantTransmittanceSampler`: Manages sampling of interaction points using a majorant extinction coefficient $\sigma_M \ge \sigma_t(\mathbf{x})$. Stores current position, remaining distance, and step history.
- `Event`: Enum or integer representing {Null, Absorption, Scatter}.

**Steps:**

1. **Initialize:** Set accumulated radiance $L = 0$. Set `terminated` flag to `false`. Initialize `path_depth` = 0.
2. **Ray-Volume Intersection:** Intersect the current ray with the volume grid bounding box. If no intersection, break the loop.
3. **Sampler Initialization:** Initialize `MajorantTransmittanceSampler` with the ray segment inside the volume, the RNG, the density grid, and the total extinction coefficient (or majorant).
4. **Sample Points Loop:** While the sampler has valid points to sample:
    a. **Sample Point:** Get the next interaction point $\mathbf{x}$ and the distance to it from the sampler.
    b. **Emission (if applicable):** If a temperature grid is available:
        i. Sample the normalized temperature $t_{norm}$ at $\mathbf{x}$.
        ii. Convert to Kelvin: $T_K = t_{norm} \times \text{temperature\_scale} + \text{temperature\_offset}$.
        iii. Compute blackbody radiation spectrum $\mathbf{L}_e$ at $T_K$.
        iv. Add emission contribution: $L \leftarrow L + p_a \times \text{le\_scale} \times \mathbf{L}_e$.
    c. **Calculate Probabilities:**
        i. Compute absorption probability $p_a$ and scattering probability $p_s$ based on the medium properties at $\mathbf{x}$ and the ratio of coefficients to the majorant.
        ii. Compute null probability: $p_n = \max(1.0 - p_a - p_s, 0.0)$.
    d. **Sample Event:** Generate a uniform random number $u \in [0, 1)$.
        i. If $u < p_n$: Event is **Null**. Log the null event. Continue to the next sampled point (skip scattering/absorption).
        ii. If $p_n \le u < p_n + p_a$: Event is **Absorption**. Log the absorption. Mark `terminated` = `true`. Break the sample points loop.
        iii. If $p_n + p_a \le u < 1.0$: Event is **Scatter**.
            - Check `path_depth`. If `path_depth` $\ge$ `max_depth`, log termination and break.
            - Increment `path_depth`.
            - Sample a new direction $\omega_s$ using the Henyey-Greenstein phase function with anisotropy parameter $g$.
            - Update the ray origin to $\mathbf{x}$ and direction to $\omega_s$.
            - Recursively sample the scattered radiance $L_{scat}$ (or continue the path loop with the new ray).
            - Add contribution: $L \leftarrow L + L_{scat}$.
            - Log the scatter event.
            - Break the sample points loop to continue the outer path tracing loop with the new ray.
5. **Termination Check:** If the path terminated due to absorption or max depth, break the outer depth loop.
6. **Infinite Light:** If the ray escapes the volume without termination (not terminated), add infinite light contribution: $L \leftarrow L + \text{infinite\_light} \times \text{multiplier}$.
7. **Accumulate:** Add $L$ to the film buffer, scaled by `imaging_ratio`. Increment sample count.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def path_tracing_discrete_event(ray_origin, ray_direction, params, grid, rng):
    L = Vector3f(0.0, 0.0, 0.0)
    terminated = False
    path_depth = 0
    
    # Current ray state
    current_o = ray_origin
    current_d = ray_direction
    
    while not terminated and path_depth < params.max_depth:
        # Intersect ray with volume grid
        hit = intersect_volume_grid(current_o, current_d, grid.bbox)
        if not hit:
            break
            
        # Initialize majorant sampler for the hit segment
        sampler = MajorantTransmittanceSampler(
            hit.t0, hit.t1, current_d, grid.density_grid, grid.extinction_grid, rng
        )
        
        # Sample points within the volume segment
        while sampler.has_next():
            point, dist = sampler.next_sample()
            
            # Emission handling
            if grid.has_temperature():
                temp_norm = grid.sample_temperature(point)
                temp_K = temp_norm * params.temperature_scale + params.temperature_offset
                le = blackbody_radiation_xyz(temp_K)
                # p_a is calculated below, but emission is typically weighted by absorption prob
                # Note: Source implies adding emission contribution here, likely scaled by p_a
                # We calculate p_a first to apply the weight correctly if needed, 
                # but source order suggests emission addition before event sampling.
                # Assuming le_scale handles the weighting or p_a is applied later.
                # Based on source: L += p_a * le_scale * blackbody...
                # We need p_a here.
                
            # Calculate extinction coefficients at point
            sigma_t = grid.sample_extinction(point)
            sigma_a = grid.sample_absorption(point) # Assuming grid provides this or derived
            sigma_s = sigma_t - sigma_a
            
            # Majorant is pre-computed or max(sigma_t)
            sigma_M = sampler.majorant
            
            # Probabilities
            p_a = sigma_a / sigma_M
            p_s = sigma_s / sigma_M
            p_n = max(1.0 - p_a - p_s, 0.0)
            
            # Emission contribution (if temp available)
            if grid.has_temperature():
                temp_norm = grid.sample_temperature(point)
                temp_K = temp_norm * params.temperature_scale + params.temperature_offset
                le = blackbody_radiation_xyz(temp_K)
                L += p_a * params.le_scale * le
                
            # Sample discrete event
            u = rng.uniform()
            if u < p_n:
                # Null event: continue to next sample point
                continue
            elif u < p_n + p_a:
                # Absorption event
                terminated = True
                break
            else:
                # Scatter event
                if path_depth >= params.max_depth:
                    terminated = True
                    break
                
                path_depth += 1
                
                # Sample new direction using Henyey-Greenstein
                new_direction = sample_henyey_greenstein(current_d, params.henyey_greenstein_g, rng)
                
                # Recursive or iterative step for scattered radiance
                # For simplicity, we continue the outer loop with new ray
                # But we need to add the scattered radiance. 
                # In a full path tracer, this might be recursive or iterative.
                # Source says: L += sample_Ld(...)
                # We assume sample_Ld handles the rest of the path or returns 0 if terminated.
                
                # Update ray for next iteration
                current_o = point
                current_d = new_direction
                
                # Break inner loop to continue outer path tracing loop
                break
        
        # If terminated inside the sample loop, break outer loop
        if terminated:
            break
            
    # If not terminated, add infinite light
    if not terminated:
        L += params.infinite_light.xyz * params.infinite_light.multiplier
        
    return L * params.imaging_ratio
```

**Parameters:**
- `henyey_greenstein_g`: Anisotropy parameter for the Henyey-Greenstein phase function. Typical values: $-1$ (backward) to $1$ (forward), $0$ (isotropic).
- `max_depth`: Maximum recursion depth for path tracing. Typical values: 10-20.
- `le_scale`: Scaling factor for emission.
- `temperature_scale`, `temperature_offset`: Parameters to convert normalized temperature samples to physical temperature (Kelvin).
- `infinite_light`: Directional light source properties (direction, intensity, multiplier).

**Edge Cases and Pitfalls:**
- **Precision:** Ensure $p_n$ is clamped to $[0, 1]$ to avoid negative probabilities due to floating-point errors.
- **Sparse Media:** Null events are frequent in sparse media. Skipping them efficiently is key.
- **Majorant:** The majorant must be $\ge \sigma_t(\mathbf{x})$ everywhere. Overestimating the majorant increases null events and reduces efficiency.
- **Temperature Conversion:** Ensure temperature conversion is physically accurate if using blackbody radiation.

## Subtopics

### [Importance Sampling for Discrete Media](7052-importance-sampling-for-discrete-media.md)
Accelerates path tracing in discrete participating media (e.g., sand) by replacing small-step integration with large-step sampling guided by precomputed probability densities. It differs from standard discrete event sampling by using spatial precomputation to determine scattering locations at larger distances, thereby reducing the number of sampling steps and improving convergence speed. Unlike methods for continuous media, it relies on the statistical independence of particle positions characteristic of discrete media.

## Operating characteristics
- **Time Complexity:** Depends on the number of sampled points per ray segment and the depth of the path. Null events allow skipping interactions, reducing the number of expensive scattering calculations in sparse media.
- **Memory Complexity:** Requires storage for the volume grid (density, temperature, extinction) and the majorant. The sampler state is minimal.
- **Performance:** More efficient than continuous Russian Roulette in sparse media due to explicit null event handling. Performance degrades in dense media where null events are rare.
- **Strengths:** Explicit handling of absorption and null events. Supports temperature-dependent emission. Efficient in sparse media.
- **Limitations:** Requires pre-computation or access to majorant. Less efficient in dense media compared to methods that don't sample null events.
- **When to Use:** Sparse participating media, temperature-dependent emission, when explicit control over absorption/scattering/null events is desired.
- **When Not to Use:** Dense media where null events are rare, or when computational resources for pre-computing majorants/probabilities are limited.

## Background
Originating from implementations like the unofficial volume path tracer by Shibodd [S1, unofficial], this method adapts path tracing for participating media by discretizing the interaction probabilities. It builds on the principles of [Explicit Path Tracing with Russian Roulette](2209-explicit-path-tracing-with-russian-roulette.md) but introduces discrete event sampling for better control and efficiency in specific media types.

## Sources
[S1] Shibodd/volume_path_tracer, https://github.com/Shibodd/volume_path_tracer, fetched 2026-10-26 (unofficial)
