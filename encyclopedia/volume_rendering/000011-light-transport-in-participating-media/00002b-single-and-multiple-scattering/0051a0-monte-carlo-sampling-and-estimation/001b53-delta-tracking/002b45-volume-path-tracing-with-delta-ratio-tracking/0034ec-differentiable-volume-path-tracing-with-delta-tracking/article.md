# Differentiable Volume Path Tracing with Delta Tracking

Also called: Differentiable Delta Tracking Path Tracer, DrTVAM Volume Integrator

## Overview
A differentiable path tracer for participating media that uses delta tracking (Woodcock tracking) to sample volumetric scattering events and handles surface interactions via BSDF sampling, accumulating separate gradients for emission, scattering, and extinction coefficients to enable inverse rendering optimization of volume parameters [S1, unofficial]. It differs from standard path tracing by maintaining distinct gradient accumulators for volume coefficients, allowing the optimization of material properties directly from image gradients [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray origin and direction, scene geometry (for surface intersection), volume texture (for coefficient lookup), random number generator state, current path depth, and active ray flags.
*   **Outputs:** Radiance contribution, updated path state, and accumulated gradients for emission, scattering, and extinction coefficients.

**Data Structures:**
The method maintains per-ray state variables:
*   `attenuation`: Float, current path throughput attenuation.
*   `δL`: Float, accumulated radiance contribution.
*   `em_grad`, `ss_grad`, `st_grad`: Floats, gradient accumulators for emission, scattering coefficient ($\sigma_s$), and extinction coefficient ($\sigma_t$) respectively.
*   `total_t`: Float, total distance traveled along the ray.
*   `n_scat`: Integer, number of scattering events.
*   `inside_target`: Boolean, flag indicating if the ray is inside the target mesh.
*   `active`: Boolean, whether the ray is still alive.
*   `depth`: Integer, current path depth.
*   `weight`: Float, current path weight (throughput).

**Parameters:**
*   `rr_depth`: Integer, Russian Roulette minimum depth. Paths shorter than this are not terminated probabilistically [S1, unofficial].
*   `max_depth`: Integer, maximum path depth. Limits the number of bounces [S1, unofficial].
*   `transmission_only`: Boolean, if true, forces BSDF sampling to only consider transmission directions [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization:** Initialize `attenuation = 1.0`, `δL = 0`, `em_grad = 0`, `ss_grad = 0`, `st_grad = 0`, `total_t = 0`, `n_scat = 0`, `inside_target = False`, `active = True`, `depth = 0` [S1, unofficial].

2.  **Russian Roulette:**
    *   Compute survival probability: $q = \min(0.99, \max(\text{attenuation}))$ [S1, unofficial].
    *   Determine if Russian Roulette should be performed: `perform_rr = (depth > rr_depth)` [S1, unofficial].
    *   Update active status: `active &= (sampler.next_1d() < q) | ~perform_rr` [S1, unofficial].
    *   Scale attenuation for terminated paths: `attenuation[perform_rr] *= 1/q` [S1, unofficial].

3.  **Surface Intersection:** Intersect ray with scene geometry: `si = scene.ray_intersect(ray, active=active)` [S1, unofficial]. Update active status: `active &= si.is_valid()` [S1, unofficial].

4.  **Medium Interaction Sampling:** Sample medium interaction using delta tracking: `mei = medium.sample_interaction(ray, sampler.next_1d(), 0, active_medium)` [S1, unofficial]. Check if surface hit occurs before medium interaction: `reached_surface = active_medium & si.is_valid() & (si.t < mei.t)` [S1, unofficial]. Set medium interaction distance to infinity if surface is hit first: `mei.t[reached_surface] = inf` [S1, unofficial].

5.  **Transmittance and PDF:**
    *   Compute transmittance and PDF: `tr, tr_pdf = medium.transmittance_eval_pdf(mei, si, active_medium)` [S1, unofficial].
    *   Compute inverse PDF: $\text{inv\_pdf} = \begin{cases} \text{detach}(1/\text{tr\_pdf}) & \text{if } \text{tr\_pdf} > 0 \\ 0 & \text{otherwise} \end{cases}$ [S1, unofficial].
    *   Update weight: `weight[active_medium] *= tr * inv_pdf` [S1, unofficial].
    *   Detach `mei.t` to prevent gradients through the distance sampling [S1, unofficial].

6.  **Coefficient Retrieval:** If no scattering occurred (purely absorptive case), set `reached_surface = active_medium` and get coefficients: `mei.sigma_s, mei.sigma_n, mei.sigma_t = medium.get_scattering_coefficients(mei, active_medium)` [S1, unofficial].

7.  **Surface BSDF Sampling:**
    *   Get BSDF: `bsdf = si.bsdf(ray)` [S1, unofficial].
    *   Sample BSDF: `bs, bs_w = bsdf.sample(ctx, si, s1, s2, active_surface & ~force_tr)` [S1, unofficial].
    *   Force transmission on first bounce if specified: `bs[force_tr], bs_w[force_tr] = bsdf.sample(...)` [S1, unofficial].
    *   Update weight: `weight[active_surface] *= bs_w` [S1, unofficial].
    *   Spawn new ray: `ray[active_surface] = si.spawn_ray(...)` [S1, unofficial].

8.  **Medium Scattering:**
    *   Update weight: `weight[active_medium] *= mei.sigma_s` [S1, unofficial].
    *   Sample phase function: `wo, phase_w = phase.sample(...)` [S1, unofficial]. Note: Phase function sampling is wrapped in `dr.suspend_grad()` to prevent gradient computation through the phase function evaluation [S1, unofficial].
    *   Update weight: `weight[active_medium] *= phase_w` [S1, unofficial].
    *   Spawn new ray: `ray[active_medium] = mei.spawn_ray(wo)` [S1, unofficial].
    *   Increment scattering count: `n_scat += 1` [S1, unofficial].

9.  **Absorption:** If no scattering occurred, apply absorption: `weight[active & reached_surface] *= exp(-mei.sigma_t * si.t)` [S1, unofficial].

10. **State Update:**
    *   Update distance: `total_t[...] += select(reached_surface, si.t, mei.t)` [S1, unofficial].
    *   Update attenuation: `attenuation[active] *= detach(weight)` [S1, unofficial].

11. **Active Flags and Depth:**
    *   Update medium/surface transition flags: `active_medium |= active_surface & (hit_target | (si.is_medium_transition() & (si.target_medium(ray.d) != None)))` [S1, unofficial].
    *   Update active status: `active &= (active_surface | active_medium)` [S1, unofficial].
    *   Increment depth: `depth[active & ~hit_target] += 1` [S1, unofficial].
    *   Terminate if max depth reached: `active &= (depth < self.max_depth)` [S1, unofficial].

12. **Gradient Propagation:** After the loop, if in backward mode, propagate gradients:
    *   `dr.backward_from(Le * em_grad)` [S1, unofficial].
    *   `dr.backward_from(sigma_t * st_grad)` [S1, unofficial].
    *   `dr.backward_from(sigma_s * ss_grad)` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def differentiable_volume_path_trace(ray, scene, medium, phase, sampler, rr_depth, max_depth, transmission_only):
    # Initialize state
    attenuation = 1.0
    delta_L = 0.0
    em_grad = 0.0
    ss_grad = 0.0
    st_grad = 0.0
    total_t = 0.0
    n_scat = 0
    inside_target = False
    active = True
    depth = 0
    weight = 1.0
    
    while active:
        # Russian Roulette
        q = min(0.99, max(attenuation))
        perform_rr = (depth > rr_depth)
        active = active and ((sampler.next_1d() < q) or not perform_rr)
        if perform_rr:
            attenuation *= 1.0 / q
            
        # Surface Intersection
        si = scene.ray_intersect(ray, active=active)
        active = active and si.is_valid()
        
        # Medium Interaction
        mei = medium.sample_interaction(ray, sampler.next_1d(), 0, active)
        reached_surface = active and si.is_valid() and (si.t < mei.t)
        if reached_surface:
            mei.t = float('inf')
            
        # Transmittance and PDF
        tr, tr_pdf = medium.transmittance_eval_pdf(mei, si, active)
        if tr_pdf > 0.0:
            inv_pdf = 1.0 / tr_pdf
        else:
            inv_pdf = 0.0
        weight *= tr * inv_pdf
        # Detach mei.t for gradient purposes
        mei_t_detached = detach(mei.t)
        
        # Coefficients
        if not n_scat: # Placeholder for "If no scattering" logic based on context
            reached_surface = active
            mei.sigma_s, mei.sigma_n, mei.sigma_t = medium.get_scattering_coefficients(mei, active)
            
        # Surface BSDF
        if si.is_valid():
            bsdf = si.bsdf(ray)
            bs, bs_w = bsdf.sample(ctx, si, sampler.next_1d(), sampler.next_1d(), active and not transmission_only)
            if transmission_only:
                bs, bs_w = bsdf.sample_transmission(ctx, si, sampler.next_1d(), sampler.next_1d())
            weight *= bs_w
            ray = si.spawn_ray(bs)
            
        # Medium Scattering
        weight *= mei.sigma_s
        with suspend_grad():
            wo, phase_w = phase.sample(mei, sampler.next_1d())
        weight *= phase_w
        ray = mei.spawn_ray(wo)
        n_scat += 1
        
        # Absorption
        if not n_scat: # Placeholder for "If no scattering"
            weight *= exp(-mei.sigma_t * si.t)
            
        # Update State
        total_t += select(reached_surface, si.t, mei_t_detached)
        attenuation *= detach(weight)
        
        # Active Flags
        active_medium = True # Placeholder for logic
        active_surface = si.is_valid()
        hit_target = False # Placeholder
        active_medium = active_medium or (active_surface and (hit_target or (si.is_medium_transition() and si.target_medium(ray.d) is not None)))
        active = active_surface or active_medium
        if not hit_target:
            depth += 1
        active = active and (depth < max_depth)
        
    # Gradient Propagation
    if backward_mode:
        dr.backward_from(Le * em_grad)
        dr.backward_from(sigma_t * st_grad)
        dr.backward_from(sigma_s * ss_grad)
        
    return delta_L
```

## Operating characteristics
*   **Time Complexity:** Proportional to the number of active rays multiplied by the average path length, which is bounded by `max_depth` [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for ray state, attenuation, and gradient accumulators, where $N$ is the number of active rays [S1, unofficial].
*   **Strengths:** Enables inverse rendering optimization of volume parameters (emission, scattering, extinction) by providing accurate gradients [S1, unofficial]. Uses delta tracking for efficient sampling in heterogeneous media [S1, unofficial].
*   **Limitations:** Purely absorptive volumes cannot be rendered with a delta tracking sensor, raising a `ValueError` if `has_scattering` is false and the sensor is `DeltaVolumetricSensor` [S1, unofficial]. Inside/outside target mesh flag flipping may cause rays to leak out [S1, unofficial].
*   **When to Use:** When optimizing volume material properties from rendered images is required.
*   **When Not to Use:** For purely absorptive volumes with delta tracking sensors, or when phase function gradients are needed (as they are suspended) [S1, unofficial].

## Background
The method is part of the Dr. TVAM (Differentiable Tomographic Volumetric Additive Manufacturing) project, which focuses on inverse rendering for tomographic volumetric additive manufacturing [S1, unofficial]. It builds on the principles of volume path tracing with delta/ratio tracking [4982-volume-path-tracing-with-delta-ratio-tracking.md] and extends them with automatic differentiation capabilities.

## Sources
[S1] Dr. TVAM – Inverse Rendering for Tomographic Volumetric Additive Manufacturing, https://github.com/rgl-epfl/drtvam, fetched 2026-10-26 (unofficial)
