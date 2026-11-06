# Volumetric Path Tracing with MIS and Surface Lighting


## Overview
This method extends volumetric path tracing to handle multiple monochromatic homogeneous volumes with absorption and multiple scattering, while integrating surface lighting via the full rendering equation [S1]. It combines phase function sampling, next event estimation (NEE), and null-scattering (delta tracking) to support heterogeneous media, using multiple importance sampling (MIS) to balance direct lighting contributions from light sources and indirect scattering [S1]. Unlike its predecessor [Next Event Estimation for Volumetric Path Tracing](1125-next-event-estimation-for-volumetric-path-tracing.md), which only added NEE, this method fully replaces surface emission $L_e$ with the rendering equation and handles chromatic media via one-sample MIS [S1].

## How it works
### Inputs and Outputs
- **Inputs**: Screen pixel coordinates, random number generator state, scene description (volumes, surfaces, lights), majorant $\sigma_m$ for each volume.
- **Output**: Radiance value for the pixel.

### Data Structures
The algorithm maintains cache variables across path bounces to resolve PDFs for MIS that may originate from previous bounces, particularly when skipping index-matching surfaces.

**Cache Variables**:
- `dir_pdf`: Float. PDF of the latest phase function sampling, measured in solid angle. Initialized to 0.
- `nee_p_cache`: Position or `None`. The last position $\mathbf{p}$ that can issue a next event estimation. If the position is on an index-matching surface, it cannot issue NEE. Initialized to `None`.
- `multi_trans_pdf`: Float. Product PDF of transmittance sampling going through several index-matching surfaces from the last phase function sampling. Initialized to 1.
- `current_path_throughput`: Float. Accumulated throughput of the path. Initialized to 1.
- `radiance`: Float. Accumulated radiance. Initialized to 0.
- `bounces`: Integer. Number of surface bounces. Initialized to 0.

### Null-Scattering and Delta Tracking
To handle heterogeneous media, the method uses null-scattering (homogenization). The radiative transfer equation is modified by adding and subtracting a term $\sigma_n(\mathbf{p}(t)) L(\mathbf{p}(t), \omega)$:

$$
\frac{d}{dt} L(\mathbf{p}(t), \omega) = -(\sigma_t(\mathbf{p}(t)) + \sigma_n(\mathbf{p}(t))) L(\mathbf{p}(t), \omega) + \sigma_n(\mathbf{p}(t)) L(\mathbf{p}(t), \omega) + \sigma_s(\mathbf{p}(t)) \int_{S^2} \rho(\omega, \omega') L(\mathbf{p}(t), \omega') d\omega'
$$

The majorant $\sigma_m$ is the upper bound of $\sigma_t(\mathbf{p}(t))$ for all $t$, ensuring $\sigma_n(\mathbf{p}(t)) = \sigma_m - \sigma_t(\mathbf{p}(t)) \ge 0$. The homogenized transmittance is $\tau_m(\mathbf{p}(0), \mathbf{p}(t')) = \exp(-\sigma_m t')$ [S1].

When sampling a distance $t_0$ from $\tau_m$, three outcomes are possible:
1. **Hit surface**: $t_0 \ge t_{\text{hit}}$. Compute $L_{\text{surface}}$.
2. **Hit real particle**: Probability $\sigma_t / \sigma_m$. Compute the scattering integral.
3. **Hit fake particle**: Probability $\sigma_n / \sigma_m$. Continue the ray in a straight line without scattering [S1].

### Exponential Integrator for NEE
For next event estimation, transmittance $\tau_p$ is estimated using an exponential integrator. The ODE for transmittance is transformed via $\tilde{\tau}_p(t') = \exp(\sigma_m t') \tau_p(t')$:

$$
\frac{d}{dt'} \tilde{\tau}_p(t') = \sigma_n(\mathbf{p}(t')) \tilde{\tau}_p(t'), \quad \tilde{\tau}_p(0) = 1
$$

This yields the recursive integral:

$$
\tau_p(t') = \exp(-\sigma_m t') + \int_0^{t'} \exp(\sigma_m(t'' - t')) \sigma_n(\mathbf{p}(t'')) \tau_p(t'') dt''
$$

This integral is solved using **ratio tracking**, a Monte Carlo sampler similar to path tracing with Russian roulette. Starting from $z = t_0$ (distance to light), we sample a distance $z_d$ such that $p(z_d) \propto \exp(\sigma_m z_d)$ and update $z$.
- If $z$ reaches 0, evaluate the light source term $\exp(-\sigma_m z)$.
- Otherwise, evaluate the recursive term. This corresponds to hitting a fake particle.
- The probability for the light source term is $\exp(-\sigma_m z)$.
- The probability density for the recursive term is $\exp(-\sigma_m z_d) / \sigma_m$ [S1].

### Chromatic Media
For RGB media, a channel (R, G, or B) is sampled whenever distance sampling is needed. The probability density is computed as the average probability density over all channels, implementing one-sample multiple importance sampling [S1].

### Algorithm Steps
1. **Initialize**: Set `current_path_throughput = 1`, `radiance = 0`, `bounces = 0`, `dir_pdf = 0`, `nee_p_cache = None`, `multi_trans_pdf = 1`.
2. **Main Loop**:
   a. **Free Flight**: Sample distance $t_0$ using homogenized transmittance $\tau_m$.
   b. **Event Handling**:
      - If $t_0 \ge t_{\text{hit}}$ (surface hit):
        - Update `nee_p_cache` if the current position is valid for NEE (not on an index-matching surface).
        - If `bounces == 0`: Add emission directly: `radiance += current_path_throughput * Le(isect)`. No MIS needed as this is the only way to see the light source [S1].
        - If `bounces > 0`: Perform NEE.
          - Set `light_point = isect`.
          - Compute `pdf_nee = pdf_point_on_light(isect.light, light_point, nee_p_cache, scene)`. Note: `pdf_nee` must account for the path vertex that issued NEE, stored in `nee_p_cache` [S1].
          - Compute MIS weight $w$. The PDF for phase function sampling is `dir_pdf * multi_trans_pdf * G`.
            $$ w = \frac{(\text{dir\_pdf} \cdot \text{multi\_trans\_pdf} \cdot G)^2}{(\text{dir\_pdf} \cdot \text{multi\_trans\_pdf} \cdot G)^2 + \text{pdf\_nee}^2} $$
          - Add radiance: `radiance += current_path_throughput * emission(vertex, -ray.dir, scene) * w`. `current_path_throughput` already accounts for transmittance [S1].
        - Sample BSDF for surface lighting (replace $L_e$ with full rendering equation) [S1].
        - Update `bounces`.
      - If real particle hit:
        - Sample scattering direction from phase function.
        - Update `dir_pdf` to the PDF of the sampled direction.
        - Update `multi_trans_pdf` if crossing index-matching surfaces.
        - Update `current_path_throughput` with scattering coefficient and phase function value.
        - Continue loop.
      - If fake particle hit:
        - Continue ray in straight line.
        - Update `multi_trans_pdf` if crossing index-matching surfaces.
        - Continue loop.
   c. **Russian Roulette**: Terminate path with probability based on throughput.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def L(screen_pos, rng):
    current_path_throughput = 1.0
    radiance = 0.0
    bounces = 0
    dir_pdf = 0.0  # in solid angle measure
    nee_p_cache = None
    multi_trans_pdf = 1.0
    
    ray = trace_primary_ray(screen_pos)
    
    while True:
        # 1. Free flight sampling using homogenized transmittance
        t0 = sample_homogenized_transmittance(ray.origin, ray.direction, sigma_m)
        
        # 2. Determine event type
        hit_surface = (t0 >= ray.length_to_surface())
        hit_real = not hit_surface and (rng.uniform() < sigma_t / sigma_m)
        
        if hit_surface:
            isect = get_surface_intersection(ray, t0)
            
            # Cache position for NEE if valid (not index-matching surface)
            if not is_index_matching_surface(isect):
                nee_p_cache = isect.position
            
            if bounces == 0:
                # Direct emission, no MIS
                radiance += current_path_throughput * Le(isect)
            else:
                # NEE with MIS
                light_point = isect.position
                pdf_nee = pdf_point_on_light(isect.light, light_point, nee_p_cache, scene)
                
                # Combined PDF for phase function sampling path
                dir_pdf_combined = dir_pdf * multi_trans_pdf * G(ray.origin, light_point) / distance_squared(ray.origin, light_point)
                
                # MIS Weight
                w = (dir_pdf_combined * dir_pdf_combined) / (dir_pdf_combined * dir_pdf_combined + pdf_nee * pdf_nee)
                
                # Add contribution
                radiance += current_path_throughput * emission(isect.vertex, -ray.direction, scene) * w
            
            # Surface lighting: Sample BSDF
            new_direction, bsdf_pdf, bsdf_value = sample_bsdf(isect, rng)
            
            # Update cache variables for next bounce
            dir_pdf = bsdf_pdf
            multi_trans_pdf = 1.0 # Reset if surface breaks continuity, or update if index-matching
            
            # Update throughput
            current_path_throughput *= bsdf_value * transmittance_to_surface(ray.origin, isect.position)
            
            ray = Ray(isect.position, new_direction)
            bounces += 1
            
        elif hit_real:
            # Hit real particle: Scatter
            new_direction, phase_pdf = sample_phase_function(ray.direction, rng)
            
            # Update cache variables
            dir_pdf = phase_pdf
            # Update multi_trans_pdf if crossing index-matching surfaces during free flight
            multi_trans_pdf *= transmittance_pdf_through_surfaces(ray.origin, ray.origin + ray.direction * t0)
            
            # Update throughput
            current_path_throughput *= sigma_s / sigma_m * phase_value(ray.direction, new_direction)
            
            ray = Ray(ray.origin + ray.direction * t0, new_direction)
            
        else:
            # Hit fake particle: Continue straight
            # Update multi_trans_pdf if crossing index-matching surfaces
            multi_trans_pdf *= transmittance_pdf_through_surfaces(ray.origin, ray.origin + ray.direction * t0)
            
            ray = Ray(ray.origin + ray.direction * t0, ray.direction)
            
        # Russian Roulette
        if rng.uniform() > current_path_throughput:
            break
        current_path_throughput /= max(current_path_throughput, 0.01) # Guard against zero
            
    return radiance
```

### Edge Cases and Pitfalls
- **Index-Matching Surfaces**: NEE cannot be issued from positions on index-matching surfaces. The `nee_p_cache` must store the last valid position [S1].
- **PDF Caching**: `dir_pdf` and `multi_trans_pdf` may refer to decisions made several bounces ago. They must be cached and updated correctly as the path progresses [S1].
- **Bounces == 0**: MIS is not needed for the first surface hit because it is the only way to see the light source [S1].
- **Chromatic Sampling**: When sampling distance for chromatic media, the PDF must be the average over all channels [S1].
- **Alternative Probability Assignment**: Using $\sigma_s / (\sigma_s + \sigma_n)$ for real/fake particle probabilities is avoided because $\sigma_s + \sigma_n$ can be 0, requiring corner case handling. The chosen method ($\sigma_t/\sigma_m$, $\sigma_n/\sigma_m$) is simpler [S1].

## Operating characteristics
- **Time Complexity**: Depends on the density of the medium and the number of bounces. Delta tracking efficiency depends on the tightness of the majorant $\sigma_m$. Ratio tracking for NEE can be expensive if $\sigma_n$ is large.
- **Memory Complexity**: Low, primarily storing cache variables and scene data.
- **Strengths**: Handles heterogeneous media via null-scattering, integrates surface lighting, and uses MIS for robust direct lighting estimation.
- **Limitations**: Requires a valid majorant $\sigma_m$. Ratio tracking for NEE can be slow if the medium is highly scattering or absorbing.
- **When to use**: Scenes with participating media and complex surfaces where direct lighting from volumes is significant.
- **When not to use**: Simple homogeneous media where closed-form solutions are sufficient, or when performance is critical and heterogeneity is low.

## Background
This method is part of a series of volumetric path tracing implementations, specifically the fifth iteration, adding surface lighting and MIS to previous versions that handled phase function sampling and NEE [S1]. It builds on the concepts of null-scattering and delta tracking introduced in earlier works [S1]. The exponential integrator and ratio tracking techniques are adapted from ODE literature and Monte Carlo integration methods [S1].

## Sources
[S1] Volumetric Path Tracing, https://cseweb.ucsd.edu/~tzli/cse272/wi2022/homework2.pdf, fetched 2026-10-30
