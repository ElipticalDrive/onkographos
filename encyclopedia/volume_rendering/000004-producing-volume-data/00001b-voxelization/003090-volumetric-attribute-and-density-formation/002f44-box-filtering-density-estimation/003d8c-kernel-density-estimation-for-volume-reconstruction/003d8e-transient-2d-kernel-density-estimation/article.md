# Transient 2D Kernel Density Estimation

Also called: Jarosz et al. 2D kernel density estimation, Temporal 2D KDE

## Overview
Transient 2D Kernel Density Estimation (KDE) estimates the radiance contribution of photon beams to a camera ray in a homogeneous participating medium by projecting the interaction onto a 2D spatial kernel and a 1D temporal window. It solves the problem of computing time-resolved light transport (transient rendering) efficiently by approximating the continuous beam contribution as a discrete density estimate. It differs from the standard 3D KDE [Kernel Density Estimation for Volume Reconstruction](9516-kernel-density-estimation-for-volume-reconstruction.md) by explicitly modeling the temporal footprint of the light pulse, and from the 1D kernel variant by distributing energy over a temporal interval rather than a single instant, introducing a trade-off between temporal bias and computational simplicity.

## How it works
**Inputs and Outputs:**
*   **Input:** Camera ray $\mathbf{r}$ defined by origin $\mathbf{x}_r$ and direction $\vec{\omega}_r$; a set of photon beams $b$ intersecting the ray; medium properties (extinction coefficient $\mu_t$, scattering coefficient $\mu_s$, index of refraction $\eta_m$); beam parameters (radius $R_b$, phase function $\Phi_{bP}$, angle $\theta_b$ between $\vec{\omega}_b$ and $\vec{\omega}_r$); intersection limits $[s_r^-, s_r^+]$ along the ray and $[s_b^-, s_b^+]$ along the beam.
*   **Output:** Radiance estimate $L_{b|2D}$ at time $t$.

**Data Structures:**
No specific complex data structures are required beyond standard scalar values for ray/beam parameters. The method operates on a per-beam basis.

**Parameters:**
*   $\eta_m$: Index of refraction of the media [S1].
*   $\mu_t$: Extinction coefficient [S1].
*   $\mu_s$: Scattering coefficient [S1].
*   $R_b$: Radius of the photon beam [S1].
*   $t_b$: Initial time of the photon beam [S1].
*   $t_r$: Initial time of the camera ray [S1].

**Steps:**
1.  **Calculate Intersection Limits:** Determine the spatial intersection intervals $[s_r^-, s_r^+]$ along the camera ray and $[s_b^-, s_b^+]$ along the photon beam.
2.  **Compute Temporal Bounds:** Calculate the start and end times of the temporal footprint using the speed of light $c$ and index of refraction $\eta_m$:
    $$ t^- = t_b + t_r + \frac{\eta_m}{c}(s_r^- + s_b^-) $$
    $$ t^+ = t_b + t_r + \frac{\eta_m}{c}(s_r^+ + s_b^+) $$
    [S1]
3.  **Evaluate Spatial Kernel:** Compute the spatial density contribution using the canonical 2D kernel $K_{2D}(R_b)$ and the geometric terms:
    $$ \text{SpatialTerm} = K_{2D}(R_b) \Phi_{bP}(\boldsymbol{\theta}_b) \mu_s \frac{e^{-\mu_t (s_c^- - s_c^+)} (|\cos \theta_b| - 1)}{e^{\mu_t (s_r^- + s_b^-)} \mu_t (|\cos \theta_b| - 1)} $$
    Note: The term $s_c^-, s_c^+$ refers to the intersection limits relevant to the specific geometric configuration described in the source, typically related to the path length within the blur region. [S1]
4.  **Evaluate Temporal Function:** Determine the temporal weight $W_{2D}(t)$ for the query time $t$:
    $$ W_{2D}(t) = \begin{cases} \frac{1}{t^+ - t^-} & \text{if } t \in (t^-, t^+) \\ 0 & \text{otherwise} \end{cases} $$
    [S1]
5.  **Combine:** Multiply the spatial term by the temporal function to get the final radiance estimate:
    $$ L_{b|2D}(\mathbf{x}_r, \boldsymbol{\omega}_r, t) = \text{SpatialTerm} \times W_{2D}(t) $$
    [S1]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def transient_2d_kde_radiance(
    ray_origin, ray_dir, beam_origin, beam_dir, beam_radius,
    mu_t, mu_s, eta_m, c, t_b, t_r, theta_b, phase_function_val,
    s_r_minus, s_r_plus, s_b_minus, s_b_plus, s_c_minus, s_c_plus,
    query_time
):
    # 1. Calculate temporal bounds
    # Note: s_r and s_b are path lengths along the ray and beam respectively
    t_minus = t_b + t_r + (eta_m / c) * (s_r_minus + s_b_minus)
    t_plus = t_b + t_r + (eta_m / c) * (s_r_plus + s_b_plus)

    # 2. Evaluate temporal function W_2D(t)
    if t_minus < query_time < t_plus:
        w_2d = 1.0 / (t_plus - t_minus)
    else:
        return 0.0

    # 3. Evaluate spatial kernel and geometric terms
    # K_2D(R_b) is the canonical 2D kernel value at radius R_b
    # This value is assumed to be precomputed or provided by the kernel implementation
    k_2d = get_2d_kernel_value(beam_radius)
    
    cos_theta_b = abs(cos(theta_b))
    
    # Numerator: Transmittance through the intersection segment and angular factor
    # Note: The source formula has a specific structure for the transmittance term
    # e^(-mu_t * (s_c_minus - s_c_plus)) suggests s_c might be a different path length
    # or a typo in the source for s_r/s_b. We follow the source formula exactly.
    transmittance_num = exp(-mu_t * (s_c_minus - s_c_plus))
    angular_factor_num = (cos_theta_b - 1.0)
    
    # Denominator: Transmittance from origin to intersection start and angular factor
    transmittance_den = exp(mu_t * (s_r_minus + s_b_minus))
    angular_factor_den = mu_t * (cos_theta_b - 1.0)
    
    # Avoid division by zero if cos_theta_b == 1
    if abs(angular_factor_den) < 1e-9:
        spatial_term = 0.0
    else:
        spatial_term = (k_2d * phase_function_val * mu_s * 
                        (transmittance_num * angular_factor_num) / 
                        (transmittance_den * angular_factor_den))

    # 4. Combine
    radiance = spatial_term * w_2d
    return radiance
```

**Edge Cases and Pitfalls:**
*   **Temporal Bias:** Evenly distributing the integrated radiance $L_b$ across the temporal interval $[t^-, t^+]$ introduces temporal bias in addition to the inherent spatial bias of density estimation. This is a known trade-off for computational efficiency [S1].
*   **Division by Zero:** If $\cos \theta_b = 1$ (perfect alignment), the term $(|\cos \theta_b| - 1)$ becomes zero, leading to division by zero in the denominator. A guard clause is required [G].
*   **Precision:** The exponential terms can underflow or overflow for large $\mu_t$ or path lengths. Standard double-precision floating point is recommended [G].

## Subtopics

### 2D Re-rendering via Kernel Density Estimation
This variant applies KDE directly in 2D image space rather than 3D volume space. Samples are re-projected from the original viewpoint to a novel viewpoint, and density is estimated in 2D. This allows rendering novel views without storing the 3D volume or more than one sample at a time, making it suitable for scenarios where the goal is simply to render images from known novel viewpoints [S2].

### Per-slice Reconstructions
This variant uses precomputed samples to reconstruct individual 2D slices through a volume on-the-fly. It is designed for interactive systems, particularly in medical imaging settings where Stochastic Tomography is applied [S2].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of intersecting photon beams $N$. Each beam is processed independently.
*   **Memory Complexity:** Low. The method does not require storing a 3D volume grid. For the 2D re-rendering variant, it requires storing only the current sample being processed [S2].
*   **Strengths:** Efficient for transient rendering in homogeneous media; allows for novel view synthesis without volumetric storage; simple temporal model (uniform distribution).
*   **Limitations:** Introduces temporal bias due to uniform energy distribution; limited to homogeneous media assumptions in the base formulation; spatial bias inherent to KDE.
*   **When to use:** When rendering transient effects in participating media where computational efficiency is critical and moderate bias is acceptable; when novel view synthesis is required without volumetric storage.
*   **When not to use:** When high temporal accuracy is required without bias; when the medium is inhomogeneous (requires extension); when full volumetric analysis is needed.

## Background
The method is an extension of the 2D kernel density estimation proposed by Jarosz et al. for steady-state volume rendering. It was adapted for transient state rendering in homogeneous media by introducing a temporal function $W_{2D}(t)$ to model the time-of-flight of light. The work is presented in "Progressive Transient Photon Beams" [S1]. The concept of 2D re-rendering via KDE is discussed in the context of Stochastic Tomography [S2].

## Sources
[S1] Progressive Transient Photon Beams, https://arxiv.org/abs/1805.09562v1, fetched 2026-10-27
[S2] Stochastic tomography and its applications in 3D imaging of mixing fluids, https://doi.org/10.1145/2185520.2185548, fetched 2026-11-03
