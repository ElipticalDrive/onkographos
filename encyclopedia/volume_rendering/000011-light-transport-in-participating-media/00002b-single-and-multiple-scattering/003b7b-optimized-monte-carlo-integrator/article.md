# Optimized Monte Carlo Integrator

Also called: Cached Monte Carlo Integration, Optimized Monte Carlo Integration, Variable Caching Integrator

## Overview
The Optimized Monte Carlo Integrator is a method for computing radiance scattered inside a participating medium from a collimated light source towards the camera, specifically designed for real-time rendering applications [S2]. It solves the problem of redundant geometric traversal by caching segment data, allowing samples to be drawn proportional to transmittance without repeated box outline calculations [S2]. It sets itself apart from standard ray marching by pre-computing analytic integration values for trapezoid-shaped segments of the medium, significantly reducing per-sample computational cost [S2].

## How it works
The method is split into two distinct phases: a geometry traversal phase that pre-computes and caches segment data, and a Monte Carlo rendering phase that selects segments and samples distances based on the cached data [S2].

### Inputs and Outputs
**Inputs:**
*   Medium geometry: A box-shaped medium with defined boundaries [S2].
*   Light source: A collimated light source (e.g., distant star or gobo-masked light) [S2].
*   Camera ray: Origin $\mathbf{p}_o$ and direction $\boldsymbol{\omega}_o$ [S2].
*   Medium properties: Extinction coefficient $\sigma_t$, scattering coefficient $\sigma_s$, and phase function $f(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o)$ [S2].
*   Number of samples: $N_{sample}$ [S2].

**Outputs:**
*   Estimated radiance $L$ at the camera pixel [S2].

### Data Structures
The core optimization relies on a cached matrix $K$ storing pre-computed values for each trapezoid-shaped segment $s$ [S2]. Each segment corresponds to a connection between an incident exit point and an outgoing exit point from the medium box [S2].

The matrix $K$ for segment $s$ contains four fields:
1.  $K_{s,1}$: Upper bound integral part, defined as $1 - E_u$ [S2].
2.  $K_{s,2}$: Slope, defined as $\sigma_t c$ (where $c$ is a geometric constant related to the segment) [S2].
3.  $K_{s,3}$: Size of the segment along the camera ray, defined as $t_{C,B} - t_{C,S}$ [S2].
4.  $K_{s,4}$: Starting position along the camera ray, defined as $t_{C,S}$ [S2].

Additionally, a cumulative distribution histogram $H$ and a multiplicative weight array $D$ are built during the setup phase [S2].

### Steps

1.  **Geometry Traversal and Caching:**
    *   Traverse the edges of the box from corner to corner to connect incident and outgoing exit points, breaking the medium geometry into trapezoid-shaped segments [S2].
    *   For each segment $k$, compute the probability of selection $G_k = T_k / T_{total}$, where $T_k$ is the transmittance contribution of segment $k$ and $T_{total}$ is the total transmittance [S2].
    *   Build a cumulative histogram $H$ by summing probabilities: $H_k \leftarrow \sum_{i=1}^k G_i$ [S2].
    *   Compute the multiplicative weight for each segment: $D_k = 1 / G_k$ [S2].
    *   Compute the summand for rescaling: $M_k \leftarrow -H_{k-1} \cdot D_k$ [S2].
    *   Cache the specific values for each segment $s$ into matrix $K$:
        *   $K_{s,1} = 1 - E_u$ [S2].
        *   $K_{s,2} = \sigma_t c$ [S2].
        *   $K_{s,3} = t_{C,B} - t_{C,S}$ [S2].
        *   $K_{s,4} = t_{C,S}$ [S2].

2.  **Monte Carlo Sampling Loop:**
    *   Initialize total visibility accumulator $V_{total} = 0$ [S2].
    *   For each sample $j$ from $1$ to $N_{sample}$:
        *   Draw a random value from a uniform distribution: $r \leftarrow \xi(k/N_{sample})$ [S2].
        *   Select the segment $s$ by comparing $r$ against the step functions of the histogram: $s \leftarrow \text{step}(H_1, r) + \text{step}(H_2, r)$ [S2]. (Note: The exact implementation of `step` implies finding the first $H_k \ge r$).
        *   Transform the sample into segment coordinates: $r_s \leftarrow r \cdot D_s + M_s$ [S2].
        *   Sample the distance $t$ along the camera ray according to the importance sampling scheme derived from the analytic integration (referencing Sec. 5.1 of the source) [S2].
        *   Accumulate visibility: $V_{total} \leftarrow V_{total} + V(\mathbf{p}_o - t \boldsymbol{\omega}_o)$ [S2]. The exponential transmittance term is cancelled by the probability density function (PDF) of the importance sampling, so only visibility $V$ is accumulated [S2].

3.  **Final Radiance Computation:**
    *   Compute the final radiance by re-weighting the total visibility:
        $$ L = \frac{1}{N_{sample}} \sigma_s f(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) V_{total} T_{total} L_i $$
        where $L_i$ is the incident radiance from the light source [S2].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimized_monte_carlo_integrator(camera_ray, medium_box, light_source, sigma_s, phase_function, N_sample):
    # Phase 1: Geometry Traversal and Caching
    segments = break_box_into_trapezoids(medium_box, camera_ray, light_source)
    
    # Compute transmittance contributions for each segment
    T_total = 0.0
    T_segments = []
    for seg in segments:
        T_k = compute_segment_transmittance(seg, camera_ray, medium_box)
        T_segments.append(T_k)
        T_total += T_k
        
    # Build Cumulative Distribution Function (CDF) histogram H
    G = [t / T_total for t in T_segments]
    H = []
    cumulative = 0.0
    for g in G:
        cumulative += g
        H.append(cumulative)
        
    # Compute weights D and M for rescaling
    D = [1.0 / g if g > 0 else 0.0 for g in G]
    M = []
    for k in range(len(H)):
        if k == 0:
            M.append(0.0) # H_{-1} is 0
        else:
            M.append(-H[k-1] * D[k])
            
    # Cache values in matrix K for each segment
    K = []
    for i, seg in enumerate(segments):
        # K_{s,1} = 1 - E_u
        # K_{s,2} = sigma_t * c
        # K_{s,3} = t_{C,B} - t_{C,S}
        # K_{s,4} = t_{C,S}
        # Note: Exact computation of E_u, c, t_{C,B}, t_{C,S} depends on geometric intersection logic
        # Not fully detailed in source, assumed available from segment geometry
        k_val = compute_segment_cache_values(seg, camera_ray, medium_box)
        K.append(k_val)
        
    # Phase 2: Monte Carlo Sampling
    V_total = 0.0
    for j in range(N_sample):
        # Draw random value
        r = random_uniform(0.0, 1.0)
        
        # Select segment s
        # Find first index k where H[k] >= r
        s = 0
        for k in range(len(H)):
            if H[k] >= r:
                s = k
                break
                
        # Transform sample in segment coordinates
        r_s = r * D[s] + M[s]
        
        # Sample distance according to importance sampling
        # This inverts the analytic integration algorithm using cached K[s]
        t = sample_distance_importance(r_s, K[s], camera_ray)
        
        # Accumulate visibility (exponential term cancelled by PDF)
        # V(p_o - t * omega_o) represents visibility/occlusion at point p_o - t * omega_o
        visibility = compute_visibility(camera_ray.origin - t * camera_ray.direction, medium_box)
        V_total += visibility
        
    # Phase 3: Final Radiance
    # L_i is incident radiance from light source
    L = (1.0 / N_sample) * sigma_s * phase_function(light_source.direction, camera_ray.direction) * V_total * T_total * light_source.intensity
    
    return L
```

### Parameters
*   $N_{sample}$: Number of Monte Carlo samples. Typical values depend on desired quality and performance constraints in real-time applications [S2].
*   Target Application: Real-time rendering [S2].

### Edge Cases and Pitfalls
*   **Zero Probability Segments:** If a segment has zero transmittance contribution ($G_k = 0$), the inverse probability $D_k$ becomes undefined. The pseudocode handles this by setting $D_k = 0$, but care must be taken to avoid division by zero during the segment selection logic if not implemented carefully [S2].
*   **Precision:** The cancellation of the exponential function by the PDF relies on the importance sampling PDF matching the transmittance distribution exactly. Numerical precision issues in computing $T_k$ or the inverse sampling function can lead to bias [S2].
*   **Non-Constant Boundary Distance:** The method assumes specific geometric properties. It will deviate in cases where the distance to the box boundary is not constant ($d_{E,S} \neq d_{E,E}$), which may require careful handling of the trapezoid segment definition [S2].

## Operating characteristics
*   **Time Complexity:** The geometry traversal phase is $O(N_{seg})$ where $N_{seg}$ is the number of trapezoid segments. The sampling phase is $O(N_{sample})$ per pixel, with segment selection being $O(\log N_{seg})$ or $O(1)$ depending on the histogram search implementation. This is significantly faster than ray marching for high sample counts because it avoids repeated geometric intersection tests for every sample [S2].
*   **Memory Complexity:** Requires storing the cached matrix $K$ and histograms $H, D, M$. The size is proportional to the number of segments $N_{seg}$ [S2].
*   **Strengths:** Delivers performance improvements over ray marching by eliminating redundant computations. Suitable for real-time rendering applications, including optical see-through AR setups [S2].
*   **Limitations:** Specifically designed for collimated light sources and box-shaped homogeneous media. The analytic integration and caching strategy is geometry-dependent and may not generalize easily to arbitrary convex shapes or inhomogeneous media without modification [S2].
*   **When to use:** Real-time rendering of volumetric light shafts from collimated sources (e.g., sunlight through windows) in homogeneous media [S2].
*   **When not to use:** Non-collimated light sources, inhomogeneous media, or media with complex non-box geometries [S2].

## Background
The method was introduced in the context of deriving an algorithm for analytic integration of unoccluded single scattering in box-shaped media [S2]. It builds on the zonal method, which was suggested by Rushmeier in a first pass to compute source radiances, followed by a Monte Carlo second pass for anisotropy [S1]. The specific "Optimized Monte Carlo Integrator" with variable caching was developed to address the computational cost of repeated box outline traversal in real-time rendering applications [S2]. It is particularly relevant for mixed reality and AR applications where additive lighting effects must be computed efficiently without obstructing the real world [S2].

## Sources
[S1] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
[S2] Collimated Whole Volume Light Scattering in Homogeneous Finite Media, https://doi.org/10.1109/tvcg.2021.3135764, fetched 2026-10-27
