# Volumetric Path Tracing with MIS and Surface Lighting
## Overview
This method extends volumetric path tracing to handle multiple monochromatic homogeneous volumes with absorption and multiple scattering, while integrating surface lighting via the full rendering equation [S1]. It combines phase function sampling, next event estimation (NEE), and null-scattering (delta tracking) to support heterogeneous media, using multiple importance sampling (MIS) to balance direct lighting contributions from light sources and indirect scattering [S1]. Unlike its predecessor [Next Event Estimation for Volumetric Path Tracing](1125-next-event-estimation-for-volumetric-path-tracing.md), which only added NEE, this method fully replaces surface emission $L_e$ with the rendering equation and handles chromatic media by averaging probability densities over sampled color channels using one-sample MIS [S1].
## How it works
### Inputs and Outputs
- **Inputs**: Screen pixel coordinates, random number generator state, scene description (volumes, surfaces, lights), and the majorant $\sigma_m$ for each volume [S1].
- **Output**: Radiance value for the pixel [S1].

### Data Structures
The algorithm maintains cache variables across path bounces to resolve PDFs for MIS that may originate from previous bounces, particularly when skipping index-matching surfaces [S1].
- `dir_pdf`: Float. PDF of the latest phase function sampling, measured in solid angle [S1]. Initialized to 0 [S1].
- `nee_p_cache`: Position or `None`. The last position $\mathbf{p}$ that can issue a next event estimation [S1]. If the position is on an index-matching surface, it cannot issue NEE [S1]. Initialized to `None` [S1].
- `multi_trans_pdf`: Float. Product PDF of transmittance sampling going through several index-matching surfaces from the last phase function sampling [S1]. Initialized to 1 [S1].
- `current_path_throughput`: Float. Accumulated throughput of the path [S1]. Initialized to 1 [S1].
- `radiance`: Float. Accumulated radiance [S1]. Initialized to 0 [S1].
- `bounces`: Integer. Number of surface bounces [S1]. Initialized to 0 [S1].

### Numbered Steps
1. **Initialize**: Set `current_path_throughput = 1`, `radiance = 0`, `bounces = 0`, `dir_pdf = 0`, `nee_p_cache = None`, `multi_trans_pdf = 1` [S1].
2. **Main Loop**:
   a. **Free Flight**: Sample distance $t_0$ using homogenized transmittance $\tau_m$ [S1].
   b. **Event Handling**:
      - If $t_0 \ge t_{\text{hit}}$ (**Hit surface**): Compute $L_{\text{surface}}$ [S1].
      - If real particle hit (**Hit real particle**): Probability $\sigma_t / \sigma_m$ [S1]. Compute the scattering integral [S1].
      - If fake particle hit (**Hit fake particle**): Probability $\sigma_n / \sigma_m$ [S1]. Continue the ray in a straight line without scattering [S1].
      - If `bounces == 0`: Add emission directly: `radiance += current_path_throughput * Le(isect)` [S1].
      - If `bounces > 0`: Perform NEE [S1]. Set `light_point = isect` [S1]. Compute `pdf_nee = pdf_point_on_light(isect.light, light_point, nee_p_cache, scene)` [S1].
      - Compute MIS weight $w$. The PDF for phase function sampling is `dir_pdf * multi_trans_pdf * G` [S1].
      - Add radiance: `radiance += current_path_throughput * emission(vertex, -ray.dir, scene) * w` [S1].
      - Sample scattering direction from phase function [S1].
      - Update `dir_pdf` to the PDF of the sampled direction [S1].
      - Update `multi_trans_pdf` if crossing index-matching surfaces [G].
      - Update `current_path_throughput` with scattering coefficient and phase function value [S1].
      - Update `bounces` [S1].
   c. **Russian Roulette**: Terminate path with probability based on throughput [G].

### Formulas
The radiative transfer equation for participating media is:
$$ \frac{d}{dt} L(\mathbf{p}(t), \omega) = -\sigma_t(\mathbf{p}(t)) L(\mathbf{p}(t), \omega) + \sigma_s(\mathbf{p}(t)) \int_{S^2} \rho(\omega, \omega') L(\mathbf{p}(t), \omega') d\omega' $$
Integrating over distance yields the volume rendering equation:
$$ L(\mathbf{p}(t), \omega) = \int_0^{t_{\text{hit}}} \tau(\mathbf{p}(0), \mathbf{p}(t')) \left( \sigma_s(\mathbf{p}(t')) \int_{S^2} \rho(\omega, \omega') L(\mathbf{p}(t'), \omega') d\omega' \right) dt' + \tau(\mathbf{p}(0), \mathbf{p}(t_{\text{hit}})) L_{\text{surface}}(\mathbf{p}(t_{\text{hit}})) $$
where transmittance is $\tau(\mathbf{p}(0), \mathbf{p}(t')) = \exp \left( - \int_0^{t'} \sigma_t(\mathbf{p}(t'')) dt'' \right)$ [S1].
For heterogeneous media, null-scattering modifies the equation by adding and subtracting $\sigma_n(\mathbf{p}(t)) L(\mathbf{p}(t), \omega)$:
$$ \frac{d}{dt} L(\mathbf{p}(t), \omega) = -(\sigma_t(\mathbf{p}(t)) + \sigma_n(\mathbf{p}(t))) L(\mathbf{p}(t), \omega) + \sigma_n(\mathbf{p}(t)) L(\mathbf{p}(t), \omega) + \sigma_s(\mathbf{p}(t)) \int_{S^2} \rho(\omega, \omega') L(\mathbf{p}(t), \omega') d\omega' $$
The homogenized transmittance is $\tau_m(\mathbf{p}(0), \mathbf{p}(t')) = \exp(-\sigma_m t')$ [S1].
For NEE, transmittance $\tau_p$ is estimated via an exponential integrator [S1]. The ODE is transformed via $\tilde{\tau}_p(t') = \exp(\sigma_m t') \tau_p(t')$:
$$ \frac{d}{dt'} \tilde{\tau}_p(t') = \sigma_n(\mathbf{p}(t')) \tilde{\tau}_p(t'), \quad \tilde{\tau}_p(0) = 1 $$
This yields the recursive integral:
$$ \tau_p(t') = \exp(-\sigma_m t') + \int_0^{t'} \exp(\sigma_m(t'' - t')) \sigma_n(\mathbf{p}(t'')) \tau_p(t'') dt'' $$
Ratio tracking solves this integral by starting at $z = t_0$, sampling $z_d$ with $p(z_d) \propto \exp(\sigma_m z_d)$, and updating $z$ [S1]. If $z$ reaches 0, evaluate the light source term $\exp(-\sigma_m z)$ [S1]. Otherwise, evaluate the recursive term [S1]. The probability for the light source term is $\exp(-\sigma_m z)$ [S1]. The probability density for the recursive term is $\exp(-\sigma_m z_d) / \sigma_m$ [S1].
MIS weight calculation uses the balance heuristic:
$$ w = \frac{(\text{dir\_pdf\_} \cdot \text{dir\_pdf\_})}{(\text{dir\_pdf\_} \cdot \text{dir\_pdf\_} + \text{pdf\_nee} \cdot \text{pdf\_nee})} $$
where `dir_pdf_` = `dir_pdf * multi_trans_pdf * G` [S1].

### Parameters
- `dir_pdf`: The PDF of the latest phase function sampling, measured in solid angle [S1].
- `nee_p_cache`: The last position $\mathbf{p}$ that can issue a next event estimation [S1].
- `multi_trans_pdf`: The product PDF of transmittance sampling going through several index-matching surfaces from the last phase function sampling [S1].
- $\sigma_m$: The majorant, defined as the upper bound of $\sigma_t(\mathbf{p}(t))$ for all $t$, ensuring $\sigma_n(\mathbf{p}(t)) \ge 0$ [S1].
- Real particle probability: $\sigma_t / \sigma_m$ [S1]. Fake particle probability: $\sigma_n / \sigma_m$ [S1].
- Probability for light source term: $\exp(-\sigma_m z)$ [S1]. Probability density for recursive term: $\exp(-\sigma_m z_d) / \sigma_m$ [S1].

### Edge Cases and Pitfalls
- **Index-Matching Surfaces**: NEE cannot be issued from positions on index-matching surfaces [S1].
- **PDF Caching**: `dir_pdf` and `multi_trans_pdf` may refer to decisions made several bounces ago [S1].
- **Bounces == 0**: MIS is not needed for the first surface hit because it is the only way to see the light source [S1].
- **Chromatic Sampling**: When sampling distance for chromatic media, the PDF must be the average over all channels [S1].
- **Alternative Probability Assignment**: Using $\sigma_s / (\sigma_s + \sigma_n)$ for real/fake particle probabilities is avoided because $\sigma_s + \sigma_n$ can be 0, requiring corner case handling [S1].

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
        u = rng.uniform()
        t0 = -math.log(u) / sigma_m

        # 2. Determine event type [S1]
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
                # G is the geometric term |cosθ|/r² [G]
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
            # Update multi_trans_pdf for index-matching surfaces crossed [G]
            multi_trans_pdf *= compute_surface_transmittance_pdf(ray_origin, ray_origin + ray_direction * t0)

            # Update throughput [S1]
            current_path_throughput *= (sigma_s / sigma_m) * phase_value(ray_direction, new_direction)

            ray_origin = ray_origin + ray_direction * t0
            ray_direction = new_direction

        else:
            # Hit fake particle: Continue straight [S1]
            # Update multi_trans_pdf for index-matching surfaces crossed [G]
            multi_trans_pdf *= compute_surface_transmittance_pdf(ray_origin, ray_origin + ray_direction * t0)

            ray_origin = ray_origin + ray_direction * t0

        # Russian Roulette [G]
        p = min(1.0, current_path_throughput)
        if rng.uniform() > p:
            break
        current_path_throughput /= p

    return radiance
```

## Operating characteristics
- **Time Complexity**: Depends on the density of the medium and the number of bounces [S1]. Delta tracking efficiency depends on the tightness of the majorant $\sigma_m$ [S1]. Ratio tracking for NEE can be expensive if $\sigma_n$ is large [S1].
- **Memory Complexity**: Low, primarily storing cache variables and scene data [S1].
- **Strengths**: Handles heterogeneous media via null-scattering, integrates surface lighting, and uses MIS for robust direct lighting estimation [S1].
- **Limitations**: Requires a valid majorant $\sigma_m$ [S1]. Ratio tracking for NEE can be slow if the medium is highly scattering or absorbing [S1].
- **When to use**: Scenes with participating media and complex surfaces where direct lighting from volumes is significant [S1].
- **When not to use**: Simple homogeneous media where closed-form solutions are sufficient, or when performance is critical and heterogeneity is low [S1].

## Background
This method is part of a series of volumetric path tracing implementations, specifically the fifth iteration, adding surface lighting and MIS to previous versions that handled phase function sampling and NEE [S1]. It builds on the concepts of null-scattering and delta tracking introduced in earlier works [S1]. The exponential integrator technique is named after a method in ODE literature [S1].

## Sources
[S1] Volumetric Path Tracing, https://cseweb.ucsd.edu/~tzli/cse272/wi2022/homework2.pdf, fetched 2026-10-30
