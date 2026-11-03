# Next Event Estimation for Volumetric Path Tracing

Also called: Direct Illumination Sampling, Direct lighting estimation, Light sampling, SampleLd, Vol Path Tracing 3, Volumetric NEE, Volumetric Path Tracing with Next Event Estimation

## Overview
Next Event Estimation (NEE) for volumetric path tracing is an optimization that estimates direct illumination in participating media by explicitly sampling light sources and tracing shadow rays through the volume and index-matched surfaces [S1][S3]. Unlike phase function sampling, which is inefficient for small light sources, NEE changes the integration domain from the hemisphere to the light surface area, accounting for transmittance via ratio-tracking and handling index-matched boundaries without terminating the shadow ray [S1]. It builds on [Multiple Scattering Volume Path Tracing with Phase Function Sampling](1043-multiple-scattering-volume-path-tracing-with-phase-function-.md) by adding direct lighting estimation to the path vertices.

## How it works

### Inputs and Outputs
**Inputs:**
*   Current scattering position $\mathbf{p}$ and view direction $\omega$ (or surface normal if at a boundary).
*   Current medium properties $\lambda$ (spectral density) and phase function $p(\omega, \omega')$.
*   Path throughput $\beta$ accumulated from the camera to $\mathbf{p}$.
*   Sampler state for random number generation.
*   Scene geometry and light sources.

**Outputs:**
*   Estimated direct illumination radiance contribution $L_{dir}(\mathbf{p}, \omega)$.

### Data Structures
*   **LightSampleContext**: Holds the interaction point and flags for surface offsetting.
*   **SampledLight**: Contains the pointer to the light source and its selection probability $p$.
*   **LightLiSample**: Contains the sampled point on the light $\mathbf{p}_{light}$, incoming direction $\omega_i$, light radiance $L$, PDF $pdf$, and light type.
*   **MediumProperties**: Contains absorption coefficient $\sigma_a$, scattering coefficient $\sigma_s$, and extinction coefficient $\sigma_t = \sigma_a + \sigma_s$.
*   **Shadow Ray State**: Variables for ratio-tracking transmittance estimation: $T_{ray}$ (current transmittance estimate), $r_l$ (lower bound PDF weight), $r_u$ (upper bound PDF weight).

### Numbered Steps

1.  **Context Initialization**:
    *   If the current vertex is a surface interaction with a BSDF, initialize `LightSampleContext` with the surface interaction.
    *   If the BSDF is reflective but not transmissive, offset the sampling origin in the direction of the outgoing vector $\omega_{out}$: $\mathbf{p}_{sample} = \mathbf{p} + \epsilon \omega_{out}$.
    *   If the BSDF is transmissive but not reflective, offset in the opposite direction: $\mathbf{p}_{sample} = \mathbf{p} - \epsilon \omega_{out}$.
    *   If the vertex is in a medium (no BSDF), use the medium interaction point directly [S3].

2.  **Light Source Sampling**:
    *   Sample a light source index $u_1$ from the sampler.
    *   Select a light source $L_{sel}$ using the light sampler's `Sample` method. If no light is selected, return 0.
    *   Sample a point on the light surface using $u_2$ (2D uniform). Get the light sample structure `ls` containing $\mathbf{p}_{light}$, $\omega_i$, $L_e$, and $pdf_{light}$.
    *   Compute the total PDF for the light selection: $pdf_{dir} = p(L_{sel}) \cdot pdf_{light}$. If $pdf_{dir} == 0$, return 0 [S3].

3.  **Scattering Function Evaluation**:
    *   Define the outgoing direction from the vertex to the light: $\omega_i = \text{normalize}(\mathbf{p}_{light} - \mathbf{p})$.
    *   If a BSDF exists:
        *   Evaluate the BSDF: $f_r = f_{bsdf}(\omega_{out}, \omega_i)$.
        *   Compute the geometry term: $G = |\omega_i \cdot \mathbf{n}|$.
        *   $f_{hat} = f_r \cdot G$.
        *   $pdf_{phase} = f_{bsdf}.PDF(\omega_{out}, \omega_i)$.
    *   If in a medium (Phase Function):
        *   Evaluate the phase function: $f_{hat} = p(\omega_{out}, \omega_i)$.
        *   $pdf_{phase} = p.PDF(\omega_{out}, \omega_i)$.
    *   If $f_{hat} == 0$, return 0 [S3].

4.  **Shadow Ray Tracing and Transmittance Estimation**:
    *   Initialize the shadow ray from $\mathbf{p}$ to $\mathbf{p}_{light}$.
    *   Initialize ratio-tracking variables: $T_{ray} = 1.0$, $r_l = 1.0$, $r_u = 1.0$.
    *   Initialize a random number generator for Russian Roulette.
    *   Loop while the ray direction is non-zero:
        1.  **Intersection**: Trace the shadow ray against the scene. Let $t_{hit}$ be the distance to the closest intersection, or $\infty$ if none.
        2.  **Opaque Occlusion**: If an intersection occurs with an opaque material (material ID $\ge 0$), the light is blocked. Return 0 [S1][S3].
        3.  **Index-Matched Surfaces**: If the intersection is with an index-matched surface (material ID $< 0$, no material):
            *   Increment a shadow bounce counter.
            *   Check max depth limits (path bounces + shadow bounces). If exceeded, return 0.
            *   Update the current medium to the medium on the other side of the surface.
            *   Continue the ray through the surface (spawn new ray segment to $\mathbf{p}_{light}$).
        4.  **Transmittance Update (Ratio-Tracking)**:
            *   Determine the segment length $t_{seg} = \min(t_{hit}, t_{max})$.
            *   If the segment is within a medium:
                *   Sample a distance $t_{sample}$ using inverse transform sampling for the exponential distribution: $t_{sample} = \frac{\ln(1-u)}{-\sigma_t}$.
                *   Calculate the majorant extinction $\sigma_{maj}$ (typically $\sigma_t$ or a precomputed upper bound).
                *   Calculate the non-absorbing part: $\sigma_n = \max(0, \sigma_{maj} - \sigma_a - \sigma_s)$.
                *   Update the transmittance estimate and PDF weights:
                    $$ T_{ray} \leftarrow T_{ray} \cdot \frac{T_{maj}(t_{seg}) \cdot \sigma_n}{T_{maj}(0) \cdot \sigma_{maj}} $$
                    $$ r_l \leftarrow r_l \cdot \frac{T_{maj}(t_{seg}) \cdot \sigma_{maj}}{T_{maj}(0) \cdot \sigma_{maj}} $$
                    $$ r_u \leftarrow r_u \cdot \frac{T_{maj}(t_{seg}) \cdot \sigma_n}{T_{maj}(0) \cdot \sigma_{maj}} $$
                    *(Note: $T_{maj}(t) = e^{-\sigma_{maj} t}$)* [S3].
                *   **Russian Roulette**: If the estimated transmittance $T_{ray} / (r_l + r_u)/2$ is very small (e.g., max component $< 0.05$), terminate with probability $q=0.75$ or scale up $T_{ray}$ by $1/(1-q)$ [S3].
        5.  **Final Segment**:
            *   If no intersection ($t_{hit} = \infty$), break the loop (light is visible).
            *   If intersection occurred (index-matched), update the ray origin and direction to the new segment towards $\mathbf{p}_{light}$.

5.  **Contribution Calculation and MIS**:
    *   If $T_{ray} == 0$, return 0.
    *   Compute the geometry term for the change of variable from solid angle to area: $G = \frac{|\omega_i \cdot \mathbf{n}_{light}|}{\|\mathbf{p}_{light} - \mathbf{p}\|^2}$.
    *   The NEE PDF (area measure) is $pdf_{nee} = pdf_{dir}$.
    *   The Phase Function PDF (solid angle measure) converted to area measure is $pdf_{phase\_area} = pdf_{phase} \cdot G$.
    *   Compute MIS weights using the power heuristic:
        $$ w = \frac{pdf_{nee}^2}{pdf_{nee}^2 + pdf_{phase\_area}^2} $$
    *   Calculate the direct lighting contribution:
        $$ L_{dir} = \beta \cdot f_{hat} \cdot T_{ray} \cdot L_e(\mathbf{p}_{light}) \cdot G $$
    *   Apply MIS weight: $L_{dir} = w \cdot L_{dir}$.
    *   Return $L_{dir}$ [S1][S3].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def next_event_estimation(p, current_medium, phase_function, beta, sampler, scene, lights):
    # 1. Context Initialization
    # Assuming intr is the interaction at p
    # If surface:
    #   ctx = LightSampleContext(intr.surface)
    #   if reflective and not transmissive:
    #       ctx.pi = intr.offset_ray_origin(intr.wo)
    #   elif transmissive and not reflective:
    #       ctx.pi = intr.offset_ray_origin(-intr.wo)
    # Else (medium):
    ctx = LightSampleContext(p) # Simplified for medium
    
    # 2. Light Source Sampling
    u1 = sampler.get_1d()
    sampled_light = lights.sample(ctx, u1)
    if not sampled_light:
        return 0.0
    
    u2 = sampler.get_2d()
    ls = sampled_light.light.sample_li(ctx, u2, sampler.get_spectral_sample())
    if not ls or ls.pdf == 0 or ls.L == 0:
        return 0.0
        
    pdf_dir = sampled_light.p * ls.pdf
    
    # 3. Scattering Function Evaluation
    wo = -normalize(p - camera_origin) # View direction
    wi = normalize(ls.p_light - p)
    
    if phase_function:
        f_hat = phase_function.p(wo, wi)
        pdf_phase = phase_function.pdf(wo, wi)
    else:
        # BSDF case (if applicable)
        f_hat = bsdf.f(wo, wi) * abs_dot(wi, bsdf.normal)
        pdf_phase = bsdf.pdf(wo, wi)
        
    if f_hat == 0:
        return 0.0
        
    # 4. Shadow Ray Tracing
    light_ray = Ray(p, ls.p_light - p)
    T_ray = 1.0
    r_l = 1.0
    r_u = 1.0
    rng = RNG(hash(light_ray.origin), hash(light_ray.direction))
    
    shadow_bounces = 0
    max_shadow_bounces = 10 # Example limit
    
    while light_ray.direction != Vec3f(0,0,0):
        isect = scene.intersect(light_ray, epsilon=1e-4)
        t_hit = isect.t if isect else float('inf')
        
        # Check for opaque occlusion
        if isect and isect.material_id >= 0:
            return 0.0
            
        # Determine segment length
        t_seg = t_hit
        
        # Update medium if index-matched surface
        if isect and isect.material_id < 0:
            shadow_bounces += 1
            if shadow_bounces > max_shadow_bounces:
                return 0.0
            current_medium = update_medium(isect, current_medium)
            # Continue ray through surface
            light_ray.origin = isect.position
            light_ray.direction = normalize(ls.p_light - light_ray.origin)
            continue
            
        # Transmittance Estimation via Ratio-Tracking
        if current_medium:
            sigma_t = current_medium.sigma_a + current_medium.sigma_s
            sigma_maj = current_medium.sigma_t # Or precomputed majorant
            
            # Sample distance
            u = rng.uniform()
            t_sample = log(1 - u) / -sigma_t
            
            # Clamp to segment
            t_sample = min(t_sample, t_seg)
            
            # Calculate T_maj for the sampled point and segment end
            T_maj_sample = exp(-sigma_maj * t_sample)
            T_maj_seg = exp(-sigma_maj * t_seg)
            
            sigma_n = max(0, sigma_maj - current_medium.sigma_a - current_medium.sigma_s)
            
            # PDF for the sampled point
            pdf_sample = T_maj_sample * sigma_maj
            
            # Update T_ray and weights
            # Note: The formula in S3 uses T_maj[0] which is 1.0 for exponential
            # T_ray *= T_maj(t_seg) * sigma_n / pdf_sample
            # This logic in S3 is slightly complex, simplifying to standard ratio tracking:
            
            # Weight for absorption/scattering part (r_l)
            # Weight for majorant remainder (r_u)
            
            # Using the specific update from S3:
            # T_ray *= T_maj * sigma_n / pdf
            # r_l *= T_maj * sigma_maj / pdf
            # r_u *= T_maj * sigma_n / pdf
            # Where T_maj and pdf are evaluated at the sampled point t_sample? 
            # S3 implies T_maj is the transmittance of the majorant over the segment?
            # Let's follow S3's lambda structure strictly:
            
            # Inside lambda for SampleT_maj:
            # sigma_n = ClampZero(sigma_maj - mp.sigma_a - mp.sigma_s)
            # pdf = T_maj[0] * sigma_maj[0] # T_maj[0] is 1.0
            # T_ray *= T_maj * sigma_n / pdf
            # r_l *= T_maj * sigma_maj / pdf
            # r_u *= T_maj * sigma_n / pdf
            
            # Here T_maj is the transmittance of the majorant over the *sampled* distance?
            # No, SampleT_maj usually returns the transmittance of the majorant over the *segment*.
            # Let's assume T_maj returned is exp(-sigma_maj * t_seg).
            
            T_maj_seg_val = exp(-sigma_maj * t_seg)
            pdf = 1.0 * sigma_maj # T_maj[0] is 1.0
            
            T_ray *= T_maj_seg_val * sigma_n / pdf
            r_l *= T_maj_seg_val * sigma_maj / pdf
            r_u *= T_maj_seg_val * sigma_n / pdf
            
            # Russian Roulette
            avg_r = (r_l + r_u) / 2.0
            if avg_r > 0:
                Tr = T_ray / avg_r
                if Tr.max_component() < 0.05:
                    q = 0.75
                    if rng.uniform() < q:
                        T_ray = 0.0
                    else:
                        T_ray /= (1 - q)
        
        # Check if ray reached light
        if t_hit == float('inf'):
            break
            
        # If index-matched, continue loop
        if isect and isect.material_id < 0:
            current_medium = update_medium(isect, current_medium)
            light_ray.origin = isect.position
            light_ray.direction = normalize(ls.p_light - light_ray.origin)
            continue
            
        # If opaque, already returned 0
        
    # 5. Final Calculation
    if T_ray == 0:
        return 0.0
        
    dist_sq = length_sq(ls.p_light - p)
    G = abs_dot(wi, ls.normal_light) / dist_sq
    
    # MIS Weights
    pdf_phase_area = pdf_phase * G
    pdf_nee = pdf_dir
    
    w = (pdf_nee * pdf_nee) / (pdf_nee * pdf_nee + pdf_phase_area * pdf_phase_area)
    
    # Contribution
    # L_dir = beta * f_hat * T_ray * L_e * G
    # Note: G is already included in the change of variable from S1 formula
    # S1 Formula: integral rho T Le |w.n|/|p-p'|^2 dp'
    # So G = |w.n|/|p-p'|^2
    
    L_dir = beta * f_hat * T_ray * ls.L * G
    
    return w * L_dir
```

### Parameters and Edge Cases
*   **Shadow Epsilon**: A small offset (e.g., $10^{-4}$) to prevent self-intersection when spawning shadow rays [S3].
*   **Max Shadow Bounces**: Limits the recursion depth for index-matched surfaces to prevent infinite loops in complex glass volumes [S1].
*   **Ratio-Tracking Majorant**: $\sigma_{maj}$ should be a tight upper bound on $\sigma_t$ to minimize Russian Roulette frequency. If $\sigma_{maj}$ is too loose, variance increases.
*   **Delta Lights**: If the light is a delta emitter (e.g., point light), the MIS weight simplifies to 1 for the NEE path, as phase function sampling cannot hit a delta light directly [S3].

## Operating characteristics

*   **Time Complexity**: $O(N \cdot M)$ where $N$ is the number of light sources and $M$ is the average number of intersections/transmittance samples per shadow ray. Ratio-tracking adds overhead proportional to the optical thickness of the medium.
*   **Memory Complexity**: $O(1)$ additional memory per shadow ray trace, excluding scene acceleration structures.
*   **Performance**: Significantly reduces variance for scenes with small, bright light sources compared to pure phase function sampling. Performance degrades in highly participating media with high optical thickness due to the cost of ratio-tracking and Russian Roulette.
*   **Strengths**: Unbiased estimation of direct lighting; handles index-matched surfaces correctly; reduces noise for small lights.
*   **Limitations**: Requires efficient shadow ray tracing; ratio-tracking can be computationally expensive in dense media; MIS weight calculation requires accurate PDFs for both methods.

## Background

Next Event Estimation for volumetric path tracing was introduced as an extension to volumetric path tracing algorithms to improve the efficiency of direct lighting estimation. The method changes the integration domain from the hemisphere to the light surface area, requiring careful handling of transmittance through participating media and index-matched boundaries [S1]. It is commonly used in modern offline renderers and research prototypes for physically based volume rendering [S3].

## Sources

[S1] PDF Volumetric Path Tracing - University of California, San Diego, https://cseweb.ucsd.edu/~tzli/cse272/wi2023/homework2.pdf, fetched 2026-10-30
[S2] GitHub - yumcyaWiz/volpt: Volumetric path tracer written in C++, https://github.com/yumcyaWiz/volpt, fetched 2026-10-27 (unofficial)
[S3] 14.2 Volume Scattering Integrators, https://pbr-book.org/4ed/Light_Transport_II_Volume_Rendering/Volume_Scattering_Integrators, fetched 2026-10-29
