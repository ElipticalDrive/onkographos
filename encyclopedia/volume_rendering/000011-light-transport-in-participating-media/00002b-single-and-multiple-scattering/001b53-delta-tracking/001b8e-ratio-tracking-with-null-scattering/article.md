# Ratio Tracking with Null Scattering

Also called: Novák et al. Transmittance Estimation, Ratio Tracking, Ratio-Tracking Transmittance Estimation

## Overview
Ratio tracking with null scattering is a Monte Carlo method for estimating transmittance in participating media that avoids the high variance of binary visibility tests by accumulating a continuous product of null-scattering ratios [S3][S5, unofficial]. Unlike standard delta tracking, which terminates a ray upon a real scattering or absorption event, ratio tracking continues sampling until the endpoint is reached, correcting the transmittance estimate at each step using the ratio of the actual extinction to the majorant extinction [S1, S4]. This technique serves as the transmittance estimation component within real-scattering vertex connection frameworks, enabling efficient handling of heterogeneous and chromatic media without binary termination outcomes [S1, S4].

## How it works

### Inputs and Outputs
**Inputs:**
*   Ray segment defined by origin $p$, direction $\omega$, and parametric bounds $[t_{min}, t_{max}]$ [S3, S4].
*   Medium properties at each point, including absorption coefficient $\sigma_a$, scattering coefficient $\sigma_s$, and extinction coefficient $\sigma_t = \sigma_a + \sigma_s$ [S2, S5, unofficial].
*   Majorant extinction coefficient $\sigma_m(x) \ge \sigma_t(x)$ for all $x$ in the medium [S1][S5, unofficial].
*   Random number generator state [S4].

**Outputs:**
*   Estimated transmittance $T$ along the ray segment [S3, S4].
*   Path throughput update factors and PDFs for Multiple Importance Sampling (MIS) [S1, S4].

### Data Structures
*   **Spectrum:** A vector of 3 floats (R, G, B) used to store transmittance, PDFs, and coefficients to support chromatic media and one-sample MIS [S1, S4].
*   **Accumulators:**
    *   $T_{ray}$: Running transmittance product, initialized to 1.0 [S4].
    *   $r_l$: Rescaled path probability for unidirectional sampling, initialized to 1.0 [S4].
    *   $r_u$: Rescaled path probability for light sampling, initialized to 1.0 [S4].
    *   $trans\_dir\_pdf$: PDF for free-flight sampling, initialized to 1.0 [S1].
    *   $trans\_nee\_pdf$: PDF for next event estimation, initialized to 1.0 [S1].

### Algorithm Steps

1.  **Initialization:**
    *   Initialize $T_{ray} = 1.0$, $r_l = 1.0$, $r_u = 1.0$ [S4].
    *   Initialize $trans\_dir\_pdf = 1.0$, $trans\_nee\_pdf = 1.0$ [S1].
    *   Determine the ray segment bounds $[t_{min}, t_{max}]$ within the medium [S3, S4].
    *   Initialize a random number generator (RNG) seeded by the ray origin and direction [S4].

2.  **Sampling Loop:**
    *   Enter an infinite loop. Sample a color channel for distance sampling by generating $u \in [0,1]$ and setting $channel = \text{clamp}(\lfloor u \times 3 \rfloor, 0, 2)$ [S1].
    *   Sample a distance $t$ from the majorant distribution for the selected channel:
        $$ t = -\frac{\log(1 - u_{rand})}{\sigma_m[channel]} $$
        where $u_{rand}$ is a uniform random number [S1, S3].
    *   Calculate the step size $dt$ relative to the remaining ray segment: $dt = t_{max} - t_{current}$ [S1, S4].
    *   Update the accumulated distance: $t_{current} = \min(t_{current} + t, t_{max})$ [S1, S4].

3.  **Event Classification and Update:**
    *   **Case A: Null Scattering (Fake Particle)** ($t < dt$):
        *   Compute the null-scattering extinction: $\sigma_n = \max(0, \sigma_m - \sigma_a - \sigma_s)$ [S4][S5, unofficial].
        *   Compute the PDF of the majorant sampling at the current point: $pdf = T_m[channel] \times \sigma_m[channel]$ [S4].
        *   Update the transmittance accumulator:
            $$ T_{ray} \leftarrow T_{ray} \times T_m \times \frac{\sigma_n}{pdf} $$
            where $T_m$ is the majorant transmittance over the step $t$ [S4].
        *   Update the rescaled path probabilities:
            $$ r_l \leftarrow r_l \times T_m \times \frac{\sigma_m}{pdf} $$
            $$ r_u \leftarrow r_u \times T_m \times \frac{\sigma_n}{pdf} $$
        *   Update the PDFs for MIS:
            $$ trans\_dir\_pdf \leftarrow trans\_dir\_pdf \times T_m \times \sigma_m \times \frac{1 - \text{real\_prob}}{\max(\sigma_m)} $$
            $$ trans\_nee\_pdf \leftarrow trans\_nee\_pdf \times T_m \times \frac{\sigma_m}{\max(\sigma_m)} $$
            where $\text{real\_prob} = \sigma_t / \sigma_m$ [S1, S4].
        *   **Numerical Stability Guard:** Divide $T_{ray}$, $r_l$, $r_u$, and the PDFs by $\max(\sigma_m)$ if a fake particle is hit to prevent overflow, as $\sigma_n$ can be large [S1, S4].
        *   **Optimization:** If $\max(T_{ray}) \le 0$, break the loop [S1].
        *   Continue to the next iteration.

    *   **Case B: Real Scattering or Endpoint Hit** ($t \ge dt$):
        *   Update the transmittance for the final segment $dt$:
            $$ T_{ray} \leftarrow T_{ray} \times T_m(dt) $$
        *   Update the PDFs:
            $$ trans\_dir\_pdf \leftarrow trans\_dir\_pdf \times T_m(dt) $$
            $$ trans\_nee\_pdf \leftarrow trans\_nee\_pdf \times T_m(dt) $$
        *   Break the loop [S1, S4].

4.  **Russian Roulette (RR):**
    *   Compute the test value for termination:
        $$ T_r = \frac{T_{ray}}{(r_l + r_u).Average()} $$
        where `.Average()` computes the mean of the RGB components [S4].
    *   If $\max(T_r) < 0.05$:
        *   Generate $u_{rr} \in [0,1]$.
        *   If $u_{rr} < q$ (where $q=0.75$), set $T_{ray} = 0$ and terminate.
        *   Else, set $T_{ray} \leftarrow T_{ray} / (1 - q)$ and continue [S4].
    *   Note: Setting the survival probability equal to transmittance would transform the method into delta tracking, resulting in binary outcomes [S4].

5.  **Final Update:**
    *   After the loop, update the accumulators for the final segment (if not already handled in Case B) by multiplying by $\frac{T_m}{T_m[0]}$ [S4].
    *   Return $T_{ray}$ as the estimated transmittance [S3, S4].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ratio_tracking_transmittance(ray, medium, t_max, rng):
    """
    Estimates transmittance along a ray segment using ratio tracking with null scattering.
    
    Args:
        ray: Ray object with origin and direction.
        medium: Medium object providing density and coefficients.
        t_max: Maximum distance to sample within the medium.
        rng: Random number generator.
        
    Returns:
        T_ray: Estimated transmittance (Spectrum).
        r_l: Rescaled path probability for unidirectional sampling (Spectrum).
        r_u: Rescaled path probability for light sampling (Spectrum).
        trans_dir_pdf: PDF for free-flight sampling (Spectrum).
        trans_nee_pdf: PDF for next event estimation (Spectrum).
    """
    # Initialization
    T_ray = Spectrum(1.0)
    r_l = Spectrum(1.0)
    r_u = Spectrum(1.0)
    trans_dir_pdf = Spectrum(1.0)
    trans_nee_pdf = Spectrum(1.0)
    
    t_current = 0.0
    max_null_collisions = 1000
    iteration = 0
    
    while True:
        # Sample a color channel for distance sampling
        u_channel = rng.uniform()
        channel = clamp(int(u_channel * 3), 0, 2)
        
        # Get majorant for the selected channel
        sigma_m = medium.get_majorant(channel)
        
        # Termination conditions
        if sigma_m <= 0 or iteration >= max_null_collisions:
            break
            
        # Sample distance from majorant distribution
        u_rand = rng.uniform()
        t = -log(1 - u_rand) / sigma_m
        
        # Calculate remaining distance
        dt = t_max - t_current
        
        # Update accumulated distance
        t_current = min(t_current + t, t_max)
        
        if t < dt:
            # Null scattering event (fake particle)
            mp = medium.get_properties(ray.position_at(t_current))
            sigma_a = mp.sigma_a[channel]
            sigma_s = mp.sigma_s[channel]
            sigma_n = max(0, sigma_m - sigma_a - sigma_s)
            
            # Majorant transmittance over step t
            T_maj = exp(-sigma_m * t)
            
            # PDF of majorant sampling
            pdf = T_maj * sigma_m
            
            # Update accumulators
            # Note: Division by max(sigma_m) is applied for numerical stability
            # The formula in [S4] uses sigma_n/pdf. 
            # We incorporate the stability normalization here.
            stability_factor = 1.0 / max(sigma_m)
            
            T_ray *= T_maj * (sigma_n / pdf) * stability_factor
            r_l *= T_maj * (sigma_m / pdf) * stability_factor
            r_u *= T_maj * (sigma_n / pdf) * stability_factor
            
            # Update PDFs for MIS
            real_prob = (mp.sigma_t[channel] / sigma_m) if sigma_m > 0 else 0
            trans_dir_pdf *= T_maj * sigma_m * (1 - real_prob) * stability_factor
            trans_nee_pdf *= T_maj * sigma_m * stability_factor
            
            # Optimization: break if transmittance is zero
            if max(T_ray) <= 0:
                break
                
            iteration += 1
            continue
            
        else:
            # Hit surface or endpoint
            # Update for the final segment dt
            T_maj_final = exp(-sigma_m * dt)
            
            T_ray *= T_maj_final
            trans_dir_pdf *= T_maj_final
            trans_nee_pdf *= T_maj_final
            
            break
            
    # Russian Roulette
    Tr_test = T_ray / (r_l + r_u).average()
    if Tr_test.max_component_value() < 0.05:
        q = 0.75
        if rng.uniform() < q:
            T_ray = Spectrum(0.0)
        else:
            T_ray /= (1 - q)
            
    return T_ray, r_l, r_u, trans_dir_pdf, trans_nee_pdf
```

### Parameters
*   `max_null_collisions`: Maximum number of iterations to prevent unbounded loops due to numerical error. Default: 1000 [S1].
*   `Termination threshold`: The maximum component value of $T_r$ below which Russian roulette is considered. Default: 0.05 [S4].
*   `q`: Russian roulette termination probability. Default: 0.75 [S4].
*   `ShadowEpsilon`: Small epsilon subtracted from 1.0 to define maximum distance for ray intersection queries, preventing self-intersection. Not given in the sources for the core algorithm, but used in scene intersection [S4].

### Edge Cases and Pitfalls
*   **Numerical Instability:** Without normalization, transmittance and PDF values can overflow because $\sigma_n$ can be large. The solution is to divide both transmittance and PDF by $\max(\sigma_m)$ whenever a fake particle is hit [S1].
*   **Chromatic Media:** PDFs must be stored as `Spectrum` (RGB) rather than `Real` to track individual channel probabilities for one-sample MIS [S1].
*   **Russian Roulette Probability:** Setting the survival probability equal to transmittance transforms the method into delta tracking, resulting in binary outcomes (0 or 1) and losing the variance reduction benefits [S4].
*   **Unbiased Estimation of Exponentials:** Direct Monte Carlo estimation of $\exp(X)$ where $X$ is an unbiased estimator of the integral is invalid because $E[\exp(X)] \neq \exp(E[X])$. Homogenization via null scattering is required to solve the recursive integral equation [S1].

## Subtopics

### [Real-scattering vertex connection with ratio tracking](1766-real-scattering-vertex-connection-with-ratio-tracking.md)
Estimates the transmittance between two points $p_1$ and $p_2$ in a participating medium using ratio tracking, which accumulates a product of null-scattering ratios rather than terminating on collision. Unlike delta tracking for sampling which terminates on collision, ratio tracking continues until the endpoint is reached, accumulating a product of ratios to estimate the total transmittance. It serves as the transmittance estimation component within the real-scattering vertex connection framework, which uses delta-tracking for path sampling.

### Homogenized Next Event Estimation (Exponential Integrator)
A method to estimate transmittance for next event estimation in heterogeneous volumes by solving an ODE derived from the transmittance definition, avoiding unbiased estimation of exponentials [S1]. It transforms the transmittance ODE $\frac{dT}{dt'} = -\sigma_t T$ into a homogenized form $\frac{d}{dt'} \tilde{T} = \sigma_n \tilde{T}$ with initial condition $\tilde{T}(0)=1$, allowing direct Monte Carlo estimation of the recursive integral equation [S1]. Unbiased estimation of $\exp(X)$ is invalid, necessitating this approach [S1]. Integrating the original ODE without homogenization is possible but Monte Carlo sampling is very slow [S1].

## Operating characteristics

*   **Time Complexity:** Proportional to the number of null scattering events sampled along the ray. The number of samples depends on the ratio of the majorant to the actual extinction coefficient. High majorants lead to more null scattering events and higher variance, requiring more samples for convergence [S3, S4].
*   **Memory Complexity:** Low, requiring only a few scalar accumulators ($T_{ray}, r_l, r_u$) and PDFs per ray segment [S4].
*   **Measured Performance:** Ratio tracking reduces variance compared to standard delta tracking, as shown in comparative images with 25 samples per pixel [S2, unofficial]. However, it increases the number of scattering events per path because paths are not terminated on absorption, leading to slower rendering times if Russian roulette is not used effectively [S2, unofficial].
*   **Strengths:** Unbiased, low-variance estimator for transmittance in heterogeneous media. Handles chromatic media efficiently via one-sample MIS. Avoids binary visibility artifacts [S3, S4].
*   **Limitations:** Can be slower than delta tracking due to increased path lengths. Requires careful tuning of Russian roulette parameters to avoid bias or inefficiency [S4].
*   **When to Use:** When low-variance transmittance estimation is critical, especially in heterogeneous or chromatic media. When used in conjunction with real-scattering vertex connection [S1, S4].
*   **When Not to Use:** When computational speed is prioritized over variance reduction and the medium is homogeneous or has a tight majorant [S2, unofficial].

## Background

Ratio tracking was introduced by Novák et al. (2014) as a variance reduction technique for transmittance estimation in participating media [S3]. It interprets the binary transmittance function as an instance of Russian roulette and replaces the random termination with a continuous multiplication of continuation probabilities [S3]. This method is a key component of null-scattering path integral formulations, which extend the Volume Rendering Equation (VRE) to handle general heterogeneous media by augmenting the medium with fictitious scattering matter [S5, unofficial]. The technique is widely used in modern unbiased volumetric path tracers, such as those described in the PBRT books [S3, S4].

## Sources

[S1] PDF Volumetric Path Tracing - University of California, San Diego, https://cseweb.ucsd.edu/~tzli/cse272/wi2023/homework2.pdf, fetched 2026-10-30
[S2] Volumetric Rendering using Path Tracing, https://zydak.github.io/Clouds/index.html, fetched 2026-10-27 (unofficial)
[S3] Light Transport II: Volume Rendering / Sampling Volume Scattering, https://www.pbr-book.org/3ed-2018/Light_Transport_II_Volume_Rendering/Sampling_Volume_Scattering.html, fetched 2026-10-28
[S4] 14.2 Volume Scattering Integrators, https://pbr-book.org/4ed/Light_Transport_II_Volume_Rendering/Volume_Scattering_Integrators, fetched 2026-10-29
[S5] A Null-Scattering Path Integral Formulation of Light Transport, https://www.scribd.com/document/1067625811/A-Null-scattering-Path-Integral-Formulation-of-Light-Transport, fetched 2026-10-29 (unofficial)
