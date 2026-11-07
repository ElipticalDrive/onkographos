# Volumetric Path Tracing with MIS and Surface Lighting
## Overview
This method extends volumetric path tracing to handle multiple monochromatic homogeneous volumes with absorption and multiple scattering, while integrating surface lighting via the full rendering equation [S1]. It combines phase function sampling, next event estimation (NEE), and null-scattering (delta tracking) to support heterogeneous media, using multiple importance sampling (MIS) to balance direct lighting contributions from light sources and indirect scattering [S1]. Unlike its predecessor [Next Event Estimation for Volumetric Path Tracing](1125-next-event-estimation-for-volumetric-path-tracing.md), which only added NEE, this method fully replaces surface emission $L_e$ with the rendering equation and handles chromatic media by averaging probability densities over sampled color channels using one-sample MIS [S1].

## How it works
### Inputs and Outputs
- **Inputs**: Screen pixel coordinates, random number generator state, scene description (volumes, surfaces, lights), majorant $\sigma_m$ for each volume [S1].
- **Output**: Radiance value for the pixel [S1].

### Data Structures
The algorithm maintains cache variables across path bounces to resolve PDFs for MIS that may originate from previous bounces, particularly when skipping index-matching surfaces [S1].
- `dir_pdf`: Float. PDF of the latest phase function sampling, measured in solid angle. Initialized to 0 [S1].
- `nee_p_cache`: Position or `None`. The last position $\mathbf{p}$ that can issue a next event estimation. If the position is on an index-matching surface, it cannot issue NEE. Initialized to `None` [S1].
- `multi_trans_pdf`: Float. Product PDF of transmittance sampling going through several index-matching surfaces from the last phase function sampling. Initialized to 1 [S1].
- `current_path_throughput`: Float. Accumulated throughput of the path. Initialized to 1 [S1].
- `radiance`: Float. Accumulated radiance. Initialized to 0 [S1].
- `bounces`: Integer. Number of surface bounces. Initialized to 0 [S1].

### Null-Scattering and Delta Tracking
To handle heterogeneous media, the method uses null-scattering (homogenization) [S1]. The radiative transfer equation is modified by adding and subtracting a term $\sigma_n(\mathbf{p}(t)) L(\mathbf{p}(t), \omega)$:
$$ \frac{d}{dt} L(\mathbf{p}(t), \omega) = -(\sigma_t(\mathbf{p}(t)) + \sigma_n(\mathbf{p}(t))) L(\mathbf{p}(t), \omega) + \sigma_n(\mathbf{p}(t)) L(\mathbf{p}(t), \omega) + \sigma_s(\mathbf{p}(t)) \int_{S^2} \rho(\omega, \omega') L(\mathbf{p}(t), \omega') d\omega' $$
The majorant $\sigma_m$ is the upper bound of $\sigma_t(\mathbf{p}(t))$ for all $t$, ensuring $\sigma_n(\mathbf{p}(t)) = \sigma_m - \sigma_t(\mathbf{p}(t)) \ge 0$ [S1]. The homogenized transmittance is $\tau_m(\mathbf{p}(0), \mathbf{p}(t')) = \exp(-\sigma_m t')$ [S1].
When sampling a distance $t_0$ from $\tau_m$, three outcomes are possible:
1. **Hit surface**: $t_0 \ge t_{\text{hit}}$. Compute $L_{\text{surface}}$ [S1].
2. **Hit real particle**: Probability $\sigma_t / \sigma_m$. Compute the scattering integral [S1].
3. **Hit fake particle**: Probability $\sigma_n / \sigma_m$. Continue the ray in a straight line without scattering [S1].

### Exponential Integrator for NEE
For next event estimation, transmittance $\tau_p$ is estimated using an exponential integrator [S1]. The ODE for transmittance is transformed via $\tilde{\tau}_p(t') = \exp(\sigma_m t') \tau_p(t')$:
$$ \frac{d}{dt'} \tilde{\tau}_p(t') = \sigma_n(\mathbf{p}(t')) \tilde{\tau}_p(t'), \quad \tilde{\tau}_p(0) = 1 $$
This yields the recursive integral:
$$ \tau_p(t') = \exp(-\sigma_m t') + \int_0^{t'} \exp(\sigma_m(t'' - t')) \sigma_n(\mathbf{p}(t'')) \tau_p(t'') dt'' $$
This integral is solved using **ratio tracking**, a Monte Carlo sampler similar to path tracing with Russian roulette [S1]. Starting from $z = t_0$ (distance to light), we sample a distance $z_d$ such that $p(z_d) \propto \exp(\sigma_m z_d)$ and update $z$ [S1].
- If $z$ reaches 0, evaluate the light source term $\exp(-\sigma_m z)$ [S1].
- Otherwise, evaluate the recursive term. This corresponds to hitting a fake particle [S1].
- The probability for the light source term is $\exp(-\sigma_m z)$ [S1].
- The probability density for the recursive term is $\exp(-\sigma_m z_d) / \sigma_m$ [S1].

### Chromatic Media
For RGB media, a channel (R, G, or B) is sampled whenever distance sampling is needed [S1]. The probability density is computed as the average probability density over all channels, implementing one-sample multiple importance sampling [S1].

### Algorithm Steps
1. **Initialize**: Set `current_path_throughput = 1`, `radiance = 0`, `bounces = 0`, `dir_pdf = 0`, `nee_p_cache = None`, `multi_trans_pdf = 1` [S1].
2. **Main Loop**:
   a. **Free Flight**: Sample distance $t_0$ using homogenized transmittance $\tau_m$ [S1].
   b. **Event Handling**:
      - If $t_0 \ge t_{\text{hit}}$ (surface hit):
        - Update `nee_p_cache` if the current position is valid for NEE (not on an index-matching surface) [S1].
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
   c. **Russian Roulette**: Terminate path with probability based on throughput [G].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def L(screen_pos, rng):
    # Initialize cache variables [S1]
    current_path_throughput = 1.0
    radiance = 0.0
    bounces = 0
    dir_pdf = 0.0  # in solid angle measure [S1]
    nee_p_cache = None
    multi_trans_pdf = 1.0

    # Generate primary ray [G]
    ray_origin, ray_direction = trace_primary_ray(screen_pos)

    while True:
        # 1. Free flight sampling using homogenized transmittance [S1]
        # Sample t0 from tau_m(t) = exp(-sigma_m * t)
        u = rng.uniform()
        t0 = -math.log(u) / sigma_m

        # 2. Determine event type
        hit_surface = (t0 >= ray_length_to_surface(ray_origin, ray_direction))
        hit_real = not hit_surface and (rng.uniform() < sigma_t / sigma_m)

        if hit_surface:
            isect = get_surface_intersection(ray_origin, ray_direction, t0)

            # Cache position for NEE if valid (not index-matching surface) [S1]
            if not is_index_matching_surface(isect):
                nee_p_cache = isect.position

            if bounces == 0:
                # Direct emission, no MIS [S1]
                radiance += current_path_throughput * Le(isect)
            else:
                # NEE with MIS [S1]
                light_point = isect.position
                pdf_nee = pdf_point_on_light(isect.light, light_point, nee_p_cache, scene)

                # Combined PDF for phase function sampling path [S1]
                dir_pdf_ = dir_pdf * multi_trans_pdf * G(ray_origin, light_point)

                # MIS Weight [S1]
                w = (dir_pdf_ * dir_pdf_) / (dir_pdf_ * dir_pdf_ + pdf_nee * pdf_nee)

                # Add contribution [S1]
                radiance += current_path_throughput * emission(isect.vertex, -ray_direction, scene) * w

            # Surface lighting: Sample BSDF [S1]
            new_direction, bsdf_pdf, bsdf_value = sample_bsdf(isect, rng)

            # Update cache variables for next bounce [S1]
            dir_pdf = bsdf_pdf
            multi_trans_pdf = 1.0  # Reset if surface breaks continuity, or update if index-matching [G]

            # Update throughput
            current_path_throughput *= bsdf_value * transmittance_to_surface(ray_origin, isect.position)

            ray_origin = isect.position
            ray_direction = new_direction
            bounces += 1

        elif hit_real:
            # Hit real particle: Scatter [S1]
            new_direction, phase_pdf = sample_phase_function(ray_direction, rng)

            # Update cache variables [S1]
            dir_pdf = phase_pdf
            # Update multi_trans_pdf if crossing index-matching surfaces during free flight [G]
            multi_trans_pdf *= transmittance_pdf_through_surfaces(ray_origin, ray_origin + ray_direction * t0)

            # Update throughput
            current_path_throughput *= (sigma_s / sigma_m) * phase_value(ray_direction, new_direction)

            ray_origin = ray_origin + ray_direction * t0
            ray_direction = new_direction

        else:
            # Hit fake particle: Continue straight [S1]
            # Update multi_trans_pdf if crossing index-matching surfaces [G]
            multi_trans_pdf *= transmittance_pdf_through_surfaces(ray_origin, ray_origin + ray_direction * t0)

            ray_origin = ray_origin + ray_direction * t0

        # Russian Roulette [G]
        p = min(1.0, current_path_throughput)
        if rng.uniform() > p:
            break
        current_path_throughput /= p

    return radiance
```

### Edge Cases and Pitfalls
- **Index-Matching Surfaces**: NEE cannot be issued from positions on index-matching surfaces. The `nee_p_cache` must store the last valid position [S1].
- **PDF Caching**: `dir_pdf` and `multi_trans_pdf` may refer to decisions made several bounces ago. They must be cached and updated correctly as the path progresses [S1].
- **Bounces == 0**: MIS is not needed for the first surface hit because it is the only way to see the light source [S1].
- **Chromatic Sampling**: When sampling distance for chromatic media, the PDF must be the average over all channels [S1].
- **Alternative Probability Assignment**: Using $\sigma_s / (\sigma_s + \sigma_n)$ for real/fake particle probabilities is avoided because $\sigma_s + \sigma_n$ can be 0, requiring corner case handling. The chosen method ($\sigma_t/\sigma_m$, $\sigma_n/\sigma_m$) is simpler [S1].

## Operating characteristics
- **Time Complexity**: Depends on the density of the medium and the number of bounces. Delta tracking efficiency depends on the tightness of the majorant $\sigma_m$. Ratio tracking for NEE can be expensive if $\sigma_n$ is large [S1].
- **Memory Complexity**: Low, primarily storing cache variables and scene data [S1].
- **Strengths**: Handles heterogeneous media via null-scattering, integrates surface lighting, and uses MIS for robust direct lighting estimation [S1].
- **Limitations**: Requires a valid majorant $\sigma_m$. Ratio tracking for NEE can be slow if the medium is highly scattering or absorbing [S1].
- **When to use**: Scenes with participating media and complex surfaces where direct lighting from volumes is significant [S1].
- **When not to use**: Simple homogeneous media where closed-form solutions are sufficient, or when performance is critical and heterogeneity is low [S1].

## Background
This method is part of a series of volumetric path tracing implementations, specifically the fifth iteration, adding surface lighting and MIS to previous versions that handled phase function sampling and NEE [S1]. It builds on the concepts of null-scattering and delta tracking introduced in earlier works [S1]. The exponential integrator technique is named after a method in ODE literature [S1].

## Sources
[S1] Volumetric Path Tracing, https://cseweb.ucsd.edu/~tzli/cse272/wi2022/homework2.pdf, fetched 2026-10-30
