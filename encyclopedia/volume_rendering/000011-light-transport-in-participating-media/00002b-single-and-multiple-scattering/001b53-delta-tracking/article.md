# Delta tracking

Also called: Null Collision Method, Null-scattering sampling, Delta Tracking with Null Collisions, delta tracking for free-path sampling

## Overview

Delta tracking is an unbiased Monte Carlo method for sampling the distance to the next interaction (scattering or absorption) in a participating medium with spatially varying (heterogeneous) extinction coefficients [S4, S9, unofficial][S16]. It solves the problem that standard exponential free-path sampling requires a constant extinction coefficient, which is rarely true in realistic volumetric scenes [S1, book][S7]. Unlike ray marching, which introduces systematic bias by discretizing the medium into fixed steps, delta tracking uses rejection sampling against a constant upper bound (majorant) to provide an exact, unbiased sample of the continuous free-path distribution [S6][S10, unofficial][S12]. It sets itself apart from siblings like [Homogeneous Medium Sampling](980-homogeneous-medium-sampling.md) by handling heterogeneity without grid constraints, and from [Ratio Tracking](1044-ratio-tracking-with-null-scattering.md) by terminating paths at real collisions rather than accumulating transmittance ratios [S3][S4, unofficial].

## How it works

**Inputs and Outputs**
*   **Input:** Ray origin $\mathbf{o}$, direction $\boldsymbol{\omega}$ (unit vector), maximum travel distance $t_{\text{max}}$ (or surface intersection distance); spatial volume data structure supporting density/extinction queries $\sigma_t(\mathbf{p})$; global or local majorant extinction coefficient $\bar{\sigma}_t$ (or $\sigma_t^{\text{max}}$) such that $\bar{\sigma}_t \ge \sigma_t(\mathbf{p})$ for all $\mathbf{p}$ along the ray [S4, S9, unofficial][S16].
*   **Output:** Distance $t$ to the first real collision, or $t_{\text{max}}$ if the ray exits the volume without interaction; boolean flag indicating if a collision occurred [S4, unofficial][S12].

**Data Structures**
*   **Volume Query Interface:** A function or method `sample_density(p)` or `get_extinction(p)` that returns the local extinction coefficient $\sigma_t(\mathbf{p})$ at position $\mathbf{p}$ [S2, book][S4, unofficial].
*   **Majorant:** A scalar value $\bar{\sigma}_t$ representing the maximum possible extinction in the queried region [S4, S9, unofficial].

**Numbered Steps**

1.  **Initialize:** Set current position $\mathbf{p} = \mathbf{o}$ and current distance traveled $t = 0$ [S4, unofficial][S12].
2.  **Sample Free Path:** Generate a uniform random number $\xi_1 \in [0, 1)$ [S4, S9, unofficial]. Sample a tentative distance $d$ from the exponential distribution defined by the majorant:
    $$ d = -\frac{\ln(1 - \xi_1)}{\bar{\sigma}_t} $$
    [S4, S9, unofficial][S12].
3.  **Check Boundary:** Update $t \leftarrow t + d$. If $t \ge t_{\text{max}}$, the ray exits the volume without a collision. Return $t_{\text{max}}$ [S4, unofficial][S12].
4.  **Evaluate Local Extinction:** Query the volume at the candidate position $\mathbf{p}_{\text{cand}} = \mathbf{o} + t \boldsymbol{\omega}$ to obtain the local extinction coefficient $\sigma_t(\mathbf{p}_{\text{cand}})$ [S4, S9, unofficial].
    *   *Pitfall:* If the query returns NaN (e.g., due to out-of-bounds access), skip this step and continue to the next iteration to avoid incorrect shading [S2, book].
5.  **Acceptance Test:** Generate a second uniform random number $\xi_2 \in [0, 1)$ [S4, S9, unofficial]. Calculate the acceptance probability:
    $$ p_{\text{accept}} = \frac{\sigma_t(\mathbf{p}_{\text{cand}})}{\bar{\sigma}_t} $$
    [S4, S9, unofficial].
    *   If $\xi_2 < p_{\text{accept}}$, the collision is **real**. Return $t$ as the distance to the first real interaction [S4, S9, unofficial].
    *   If $\xi_2 \ge p_{\text{accept}}$, the collision is a **null collision** (virtual particle). The ray passes through without interaction. Continue to Step 2 from the current position $\mathbf{p}_{\text{cand}}$ [S4, S9, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def delta_tracking(ray_origin, ray_direction, t_max, majorant_sigma_t, volume_query):
    """
    Samples the distance to the first real collision in a heterogeneous medium.
    
    Args:
        ray_origin (vec3): Origin of the ray.
        ray_direction (vec3): Unit direction vector of the ray.
        t_max (float): Maximum distance to travel (e.g., to surface or volume boundary).
        majorant_sigma_t (float): Upper bound on extinction coefficient (sigma_t >= 0).
        volume_query (function): Callable that takes a position (vec3) and returns 
                                 the local extinction coefficient (float).
    
    Returns:
        float: Distance to the first real collision. 
               Returns t_max if no collision occurs within the limit.
    """
    current_t = 0.0
    
    while True:
        # Step 2: Sample tentative distance from majorant distribution
        xi1 = random_uniform(0.0, 1.0)
        # Avoid log(0) by using 1-xi1 which is in (0, 1]
        tentative_dist = -log(1.0 - xi1) / majorant_sigma_t
        
        current_t += tentative_dist
        
        # Step 3: Check if ray exits volume
        if current_t >= t_max:
            return t_max
        
        # Calculate candidate position
        candidate_pos = ray_origin + ray_direction * current_t
        
        # Step 4: Evaluate local extinction
        local_sigma_t = volume_query(candidate_pos)
        
        # Pitfall handling: Skip if query fails (e.g., NaN)
        if isnan(local_sigma_t):
            continue
            
        # Step 5: Acceptance/Rejection test
        xi2 = random_uniform(0.0, 1.0)
        acceptance_prob = local_sigma_t / majorant_sigma_t
        
        if xi2 < acceptance_prob:
            # Real collision found
            return current_t
        else:
            # Null collision: continue sampling from new position
            pass
```

**Parameters**
*   **Majorant ($\bar{\sigma}_t$):** Must be a strict upper bound of the local extinction coefficient everywhere along the ray [S4, unofficial][S16]. If $\bar{\sigma}_t$ is too loose (much larger than local $\sigma_t$), the acceptance probability drops, leading to many null collisions and poor performance [S9, S11, unofficial].
*   **Random Numbers:** $\xi_1, \xi_2$ are independent uniform random variables in $[0, 1)$ [S4, S9, unofficial].

**Edge Cases and Pitfalls**
*   **Loose Majorant:** In volumes with large empty spaces, a global majorant leads to excessive null collisions [S9, S11, unofficial]. Use local majorants or subdivided regions to tighten the bound [S1, book][S14, unofficial].
*   **NaN Values:** Volume queries may return NaN for out-of-bounds or invalid data. The algorithm must handle this gracefully by skipping the rejection test and continuing [S2, book].
*   **Precision:** If `local_sigma_t` exceeds `majorant_sigma_t` due to floating-point errors or incorrect bounds, `acceptance_prob` > 1, breaking the logic. Ensure `majorant_sigma_t` is strictly $\ge$ all possible `local_sigma_t` values [S4, unofficial].
*   **Infinite Loops:** If the volume has infinite extent or $t_{\text{max}}$ is not properly bounded, the loop may not terminate. Always enforce a maximum travel distance [S4, unofficial][S12].

## Subtopics

### [Ratio Tracking with Null Scattering](1044-ratio-tracking-with-null-scattering.md)
Continues sampling along the ray, multiplying transmittance estimates by the ratio of actual to majorant extinction, avoiding binary visibility tests. Choose when low-variance transmittance estimation is needed for heterogeneous media.

### [Woodcock Tracking with Variable Majorants](1084-woodcock-tracking-with-variable-majorants.md)
Uses spatially varying upper bounds (e.g., grid macrocells) to reduce null collisions in sparse regions. Choose when the volume has significant heterogeneity and empty spaces.

### [Delta Tracking (Uniform Majorant)](1539-delta-tracking-uniform-majorant.md)
Simplified version assuming a uniform majorant density of 1.0. Choose for homogeneous media or when simplicity outweighs efficiency in heterogeneous cases.

### [Null Interactions Shift](1836-null-interactions-shift.md)
Bijective transformation to map interaction depths between camera viewpoints. Choose for multi-view rendering and sample reuse.

### [Volume Path Tracing with Delta/Ratio Tracking](4982-volume-path-tracing-with-delta-ratio-tracking.md)
Combines delta tracking for scattering events and ratio tracking for transmittance in WGSL. Choose for GPU-based path tracing with distinct emission/absorption handling.

### [Weighted Delta Tracking](9238-weighted-delta-tracking.md)
Relaxes the majorant constraint, allowing $\bar{\mu} < \mu_t$ at the cost of statistical weights. Choose when simpler sampling distributions are preferred over variance control.

### [Delta Tracking with Volume State Changes](11135-delta-tracking-with-volume-state-changes.md)
Handles temporal or iterative density changes by bounding both current and previous states. Choose for inverse rendering or animation.

## Operating characteristics

*   **Time Complexity:** Expected number of steps is proportional to the ratio of the majorant to the local extinction coefficient. In homogeneous regions, it converges quickly. In sparse regions with a loose global majorant, it can be $O(\bar{\sigma}_t / \sigma_t)$, which is inefficient [S9, S11, unofficial].
*   **Memory Complexity:** $O(1)$ per ray, excluding the volume data structure storage. It does not require storing intermediate samples along the ray [S1, book][S12].
*   **Measured Performance:** On a 51 million particle HACC dataset, delta tracking path tracing achieved 1.44 MR/s, compared to 6.0 MR/s for ray casting, but with higher quality and unbiased results [S2, book]. Rendering a cloud with delta tracking and Next Event Estimation (10 samples/pixel) took 248s [S9, unofficial].
*   **Strengths:** Unbiased; handles arbitrary heterogeneity without grid constraints; simple implementation; no fixed step size bias [S6][S10, unofficial][S12].
*   **Limitations:** High variance in sparse regions with loose majorants; requires efficient volume queries; many null collisions waste computation [S9, S11, unofficial].
*   **When to Use:** Unbiased path tracing in heterogeneous media; particle volumes; when grid-based methods are impractical [S2, book][S12].
*   **When Not to Use:** Real-time rendering with strict latency constraints in sparse volumes (use ray marching or ratio tracking with optimizations); homogeneous media (use direct exponential sampling) [S9, unofficial][S12].

## Background

Delta tracking, also known as the null-collision method or Woodcock tracking, was developed independently in the 1950s and 1960s for neutron transport and plasma physics [S6][S14, unofficial]. Key contributors include Butcher and Messel (1958, 1960), Zerby et al. (1961), Bertini (1963), Woodcock et al. (1965), Miller (1967), and Coleman (1968) [S14, unofficial]. It was adopted in computer graphics for unbiased volume rendering to avoid the bias of ray marching [S6][S10, unofficial]. It is implemented in systems like OSPRay using ISPC and Open VKL for RBF particle volumes [S2, book].

## Sources

[S1] Efficient Unbiased Volume Path Tracing on the GPU, https://doi.org/10.1007/978-1-4842-7185-8_43, fetched 2026-10-24 (book)
[S2] Path Tracing RBF Particle Volumes, https://link.springer.com/chapter/10.1007/978-1-4842-7185-8_44, fetched 2026-10-25 (book)
[S3] Real-Time Denoising of Volumetric Path Tracing for Direct Volume Rendering, https://doi.org/10.1109/tvcg.2020.3037680, fetched 2026-10-26
[S4] ainagutierrez01/Volumetric-Path-Tracing-, https://github.com/ainagutierrez01/Volumetric-Path-Tracing-, fetched 2026-10-26 (unofficial)
[S5] shamanDevel/fV-SRN, https://github.com/shamanDevel/fV-SRN, fetched 2026-10-26 (unofficial)
[S6] Pseudo-marginal metropolis light transport, https://doi.org/10.1145/2820903.2820922, fetched 2026-10-30
[S7] PDF Volumetric Path Tracing - University of California, San Diego, https://cseweb.ucsd.edu/~tzli/cse272/wi2023/homework2.pdf, fetched 2026-10-30
[S8] GitHub - yumcyaWiz/volpt: Volumetric path tracer written in C++, https://github.com/yumcyaWiz/volpt, fetched 2026-10-27 (unofficial)
[S9] Volumetric Rendering using Path Tracing, https://zydak.github.io/Clouds/index.html, fetched 2026-10-27 (unofficial)
[S10] PDF Volumetric Path Tracing, https://jugoodma.github.io/docs/740-paper.pdf, fetched 2026-10-27 (unofficial)
[S11] Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data, https://repository.bilkent.edu.tr/items/fcbc592a-2646-49af-8b84-67308b6fd3e9, fetched 2026-10-27 (unofficial)
[S12] Light Transport II: Volume Rendering / Sampling Volume Scattering, https://www.pbr-book.org/3ed-2018/Light_Transport_II_Volume_Rendering/Sampling_Volume_Scattering.html, fetched 2026-10-28
[S13] 14.2 Volume Scattering Integrators, https://pbr-book.org/4ed/Light_Transport_II_Volume_Rendering/Volume_Scattering_Integrators, fetched 2026-10-29
[S14] A Null-Scattering Path Integral Formulation of Light Transport, https://www.scribd.com/document/1067625811/A-Null-scattering-Path-Integral-Formulation-of-Light-Transport, fetched 2026-10-29 (unofficial)
[S15] Volume Rendering: Implementation, https://computergraphics.stackexchange.com/questions/5220/volume-rendering-implementation?noredirect=1&lq=1, fetched 2026-10-30 (unofficial)
[S16] PDF Spectral and Decomposition Tracking for Rendering Heterogeneous Volumes, https://media.disneyanimation.com/uploads/production/publication_asset/158/asset/SpectralAndDecompositionTracking.pdf, fetched 2026-10-30
