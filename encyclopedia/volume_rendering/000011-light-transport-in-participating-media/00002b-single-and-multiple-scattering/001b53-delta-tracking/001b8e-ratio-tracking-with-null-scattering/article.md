# Ratio Tracking with Null Scattering

Also called: Novák et al. Transmittance Estimation, Ratio Tracking, Ratio-Tracking Transmittance Estimation

## Overview
Ratio tracking is a Monte Carlo variance reduction technique for estimating transmittance in participating media, used primarily within null-scattering frameworks to handle heterogeneous volumes [S1, S3]. Unlike standard delta tracking, which terminates a path with a binary outcome (absorption or scattering) upon collision, ratio tracking continues sampling until the endpoint is reached, accumulating a product of ratios to estimate the total transmittance [S1][S2, unofficial]. This approach avoids the high variance associated with binary visibility tests while maintaining unbiasedness [S3]. It builds on [Delta tracking](985-delta-tracking.md) by replacing the binary Russian roulette termination with a continuous weight update [S3].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin $o$, direction $\omega$, maximum distance $t_{max}$ (determined by scene intersection or light source), medium properties (extinction $\sigma_t$, absorption $\sigma_a$, scattering $\sigma_s$, or albedo $a$), and a majorant extinction coefficient $\sigma_m \ge \sigma_t(x)$ for all $x$ in the medium [S1, S4].
*   **Output:** Estimated transmittance $T$ (a Spectrum), and PDFs for Multiple Importance Sampling (MIS) if required [S1, S4].

**Data Structures:**
*   **Spectrum:** A vector of 3 floats (R, G, B) used to store transmittance and PDFs to support chromatic media handling [S1].
*   **Majorant ($\sigma_m$):** A constant or spatially varying upper bound on the extinction coefficient. For chromatic media, $\sigma_m$ is typically a Spectrum where each component is the maximum extinction in that channel [S1].
*   **Null-scattering coefficient ($\sigma_n$):** Defined as $\sigma_n(x) = \max(0, \sigma_m - \sigma_t(x))$ [S4].

**Parameters:**
*   `max_null_collisions`: Maximum iterations to prevent infinite loops due to numerical error. Default: 1000 [S1].
*   `termination_threshold`: Threshold for Russian roulette. Default: 0.05 [S4].
*   `q`: Russian roulette termination probability. Default: 0.75 [S4].
*   `ShadowEpsilon`: Small epsilon subtracted from 1.0 to define max distance, preventing self-intersection [S4].

**Steps:**
1.  **Initialize:** Set transmittance accumulator $T_{ray} = 1.0$, rescaled path probabilities $r_l = 1.0$ (for unidirectional sampling) and $r_u = 1.0$ (for light sampling), and PDFs `trans_dir_pdf` and `trans_nee_pdf` to 1.0 [S1, S4]. Initialize RNG with hash of ray origin and direction [S4].
2.  **Determine Bounds:** Compute $t_{min}$ and $t_{max}$ for the ray segment within the medium. If no surface intersection is found, $t_{max}$ is determined by the light source distance minus `ShadowEpsilon` [S4].
3.  **Sample Distance:** Sample a distance $t$ from the majorant distribution. For chromatic media, sample a single color channel (R, G, or B) using a uniform random number $u \in [0,1]$ to select the channel, then sample $t = -\log(1-u)/\sigma_m[\text{channel}]$ [S1].
4.  **Check Endpoint:** If $t \ge t_{max}$ (or accumulated distance reaches $t_{max}$), update transmittance for the final segment and break the loop [S1, S4].
5.  **Evaluate Null Scattering:** At the sampled point, compute the null-scattering coefficient $\sigma_n = \max(0, \sigma_m - \sigma_a - \sigma_s)$ [S4].
6.  **Update Accumulators:**
    *   Compute the PDF of the majorant sampling at the point: $pdf = T_{maj}[0] \times \sigma_{maj}[0]$ [S4].
    *   Update transmittance: $T_{ray} *= T_{maj} \times \frac{\sigma_n}{pdf}$ [S4].
    *   Update rescaled probabilities: $r_l *= T_{maj} \times \frac{\sigma_{maj}}{pdf}$ and $r_u *= T_{maj} \times \frac{\sigma_n}{pdf}$ [S4].
    *   Update PDFs for MIS: `trans_dir_pdf` and `trans_nee_pdf` are updated similarly to account for the decision to continue (null scattering) vs. terminate (real scattering) [S1, S4].
    *   **Normalization:** Divide $T_{ray}$ and PDFs by $\max(\sigma_m)$ if a null scattering event occurs to prevent numerical blow-up [S1].
7.  **Russian Roulette:** Compute test value $Tr = T_{ray} / (r_l + r_u).Average()$. If $Tr.MaxComponentValue() < 0.05$, terminate with probability $q=0.75$ (setting $T_{ray}=0$) or survive with probability $1-q$ (dividing $T_{ray}$ by $1-q$) [S4].
8.  **Finalize:** After the loop, update accumulators for the final segment from the last sampled point to $t_{max}$ by multiplying by $\frac{T_{maj}}{T_{maj}[0]}$ [S4]. Return $T_{ray}$ [S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ratio_tracking(ray, medium, rng):
    # 1. Initialize
    t_min, t_max = compute_bounds(ray, medium)
    if t_max <= t_min:
        return Spectrum(1.0)
    
    T_ray = Spectrum(1.0)
    r_l = Spectrum(1.0)
    r_u = Spectrum(1.0)
    trans_dir_pdf = Spectrum(1.0)
    trans_nee_pdf = Spectrum(1.0)
    
    sigma_m = medium.get_majorant() # Spectrum
    inv_max_density = 1.0 / sigma_m
    
    # 2. Loop
    while True:
        # 3. Sample distance
        # Sample a channel for chromatic media
        u = rng.uniform()
        channel = clamp(int(u * 3), 0, 2)
        
        if sigma_m[channel] <= 0:
            break
            
        t_step = -log(1 - rng.uniform()) * inv_max_density[channel]
        
        # 4. Check endpoint
        if t_min + t_step >= t_max:
            dt = t_max - t_min
            # Update for final segment
            T_ray *= exp(-sigma_m * dt)
            trans_dir_pdf *= exp(-sigma_m * dt)
            trans_nee_pdf *= exp(-sigma_m * dt)
            break
            
        t_min += t_step
        point = ray.origin + ray.direction * t_min
        
        # 5. Evaluate Null Scattering
        sigma_t = medium.get_extinction(point)
        sigma_a = medium.get_absorption(point)
        sigma_s = medium.get_scattering(point)
        sigma_n = clamp_zero(sigma_m - sigma_a - sigma_s)
        
        # 6. Update Accumulators
        # PDF of majorant sampling
        pdf = T_ray[0] * sigma_m[0] # Note: T_ray[0] is T_maj[0] from context, usually 1.0 at start of segment
        
        # Correction factor
        # In many implementations, T_maj is the transmittance of the majorant from start to current point
        # Here we assume T_maj is passed or computed. 
        # Simplified: T_maj = exp(-sigma_m * t_step) relative to previous step? 
        # The source [S4] uses SampleT_maj which returns T_maj for the segment.
        # We approximate T_maj for the step as exp(-sigma_m * t_step)
        T_maj_step = exp(-sigma_m * t_step)
        
        # Update T_ray
        # T_ray *= T_maj * sigma_n / pdf
        # Note: pdf in source is T_maj[0] * sigma_maj[0]. 
        # If we treat the sampling distribution as p(t) = sigma_m * exp(-sigma_m * t),
        # then the ratio is sigma_n / sigma_m.
        # The source [S4] formula: T_ray *= T_maj * sigma_n / pdf.
        # Let's follow the source's variable names strictly.
        
        # Assuming T_maj is the transmittance of the majorant over the sampled distance t_step
        # pdf = T_maj[0] * sigma_maj[0] is likely a typo in notes for T_maj * sigma_m? 
        # Actually, p(t) = sigma_m * exp(-sigma_m * t). 
        # The term T_maj * sigma_n / pdf becomes exp(-sigma_m * t) * sigma_n / (sigma_m * exp(-sigma_m * t)) = sigma_n / sigma_m.
        # This matches the ratio tracking concept.
        
        ratio = sigma_n / sigma_m
        T_ray *= T_maj_step * ratio
        r_l *= T_maj_step * (sigma_m / pdf) # r_l tracks real scattering prob
        r_u *= T_maj_step * ratio # r_u tracks null scattering prob
        
        # Normalize to prevent blow-up
        T_ray /= max(sigma_m)
        trans_dir_pdf /= max(sigma_m)
        trans_nee_pdf /= max(sigma_m)
        
        # 7. Russian Roulette
        Tr = T_ray / (r_l + r_u).average()
        if Tr.max_component_value() < 0.05:
            q = 0.75
            if rng.uniform() < q:
                T_ray = Spectrum(0.0)
                break
            else:
                T_ray /= (1 - q)
                
    # 8. Finalize
    # Update for final segment if loop broke early or completed
    # Source [S4] updates after SampleT_maj returns
    T_ray *= sigma_m / sigma_m[0] # This line in source seems to handle the last segment's majorant transmittance normalization
    r_l *= sigma_m / sigma_m[0]
    r_u *= sigma_m / sigma_m[0]
    
    return T_ray
```

**Edge Cases and Pitfalls:**
*   **Numerical Instability:** Without normalization, transmittance and PDF values can blow up because $\sigma_n$ can be large. Solution: Divide both transmittance and PDF by $\max(\sigma_m)$ whenever a fake particle is hit [S1].
*   **Russian Roulette Choice:** If the survival probability were set equal to transmittance, the method would effectively become delta tracking, resulting in a transmittance estimate that is always either 0 or 1, losing the benefits of ratio tracking's continuous estimation [S4].
*   **Chromatic Media:** PDFs must be stored as `Spectrum` types to track individual channel probabilities for MIS. The final PDF used for estimation is the average of the channel PDFs [S1].
*   **Infinite Loops:** Numerical errors can lead to unbounded loops. A maximum iteration count (`max_null_collisions`) is required [S1].

## Subtopics

### [Real-scattering vertex connection with ratio tracking](1766-real-scattering-vertex-connection-with-ratio-tracking.md)
Estimates the transmittance between two points $p_1$ and $p_2$ in a participating medium using ratio tracking, which accumulates a product of null-scattering ratios rather than terminating on collision. Unlike delta tracking for sampling which terminates on collision, ratio tracking continues until the endpoint is reached, accumulating a product of ratios to estimate the total transmittance. This method serves as the transmittance estimation component within the broader real-scattering vertex connection framework, where it connects real-scattering vertices from camera and light subpaths.

### Homogenized Next Event Estimation (Exponential Integrator)
A method to estimate transmittance for next event estimation in heterogeneous volumes by solving an ODE derived from the transmittance definition, avoiding unbiased estimation of exponentials [S1]. It transforms the transmittance ODE $\frac{dT}{dt'} = -\sigma_t T$ into $\frac{d}{dt'} \tilde{T} = \sigma_n \tilde{T}$ via change of variable $\tilde{T} = e^{\sigma_m t} T$, allowing Monte Carlo estimation of the integral without exponentiation [S1]. Unbiased estimation of $\exp(X)$ is invalid because $E[\exp(X)] \neq \exp(E[X])$, necessitating this homogenization approach [S1]. Integrating both sides of the original ODE yields a slow-converging integral, making the homogenized version preferred [S1].

## Operating characteristics
*   **Time Complexity:** Proportional to the number of null scattering events sampled. In highly heterogeneous media with large majorants, the number of null events can be high, increasing computational cost compared to homogeneous media [S2, unofficial].
*   **Memory Complexity:** Low. Requires storing only the current transmittance, PDFs, and medium properties at sampled points [S1, S4].
*   **Measured Performance:** Reduces variance compared to standard delta tracking, as shown in comparative images with 25 samples per pixel [S2, unofficial]. However, it increases the number of scattering events per path because paths are not terminated on absorption, leading to slower rendering times in some scenarios [S2, unofficial].
*   **Strengths:** Unbiased, low-variance estimator for transmittance in heterogeneous media [S3]. Handles spectral variations and MIS weights efficiently [S4]. Avoids binary visibility issues [S1].
*   **Limitations:** Slower than delta tracking in terms of raw throughput due to increased path lengths [S2, unofficial]. Requires careful handling of numerical stability [S1].
*   **When to Use:** When low variance is critical and the medium is heterogeneous. When using null-scattering path tracing frameworks [S5, unofficial].
*   **When Not to Use:** When performance is paramount and high variance is acceptable (use standard delta tracking). When the medium is homogeneous (analytical transmittance is available).

## Background
Ratio tracking was introduced by Novák et al. in 2014 as a method to solve the recursive integral equation for transmittance in participating media [S3]. It was developed to address the high variance of binary Russian roulette estimators used in standard delta tracking [S3]. The method is a key component of null-scattering formulations, which augment the medium with fictitious scattering matter to enable analytic sampling of transmittance in heterogeneous volumes [S5, unofficial]. It is widely used in modern volumetric path tracers, such as those described in the PBRT books and various academic implementations [S1, S4].

## Sources
[S1] PDF Volumetric Path Tracing - University of California, San Diego, https://cseweb.ucsd.edu/~tzli/cse272/wi2023/homework2.pdf, fetched 2026-10-30
[S2] Volumetric Rendering using Path Tracing, https://zydak.github.io/Clouds/index.html, fetched 2026-10-27 (unofficial)
[S3] Light Transport II: Volume Rendering / Sampling Volume Scattering, https://www.pbr-book.org/3ed-2018/Light_Transport_II_Volume_Rendering/Sampling_Volume_Scattering.html, fetched 2026-10-28
[S4] 14.2 Volume Scattering Integrators, https://pbr-book.org/4ed/Light_Transport_II_Volume_Rendering/Volume_Scattering_Integrators, fetched 2026-10-29
[S5] A Null-Scattering Path Integral Formulation of Light Transport, https://www.scribd.com/document/1067625811/A-Null-scattering-Path-Integral-Formulation-of-Light-Transport, fetched 2026-10-29 (unofficial)
