# Biased Sampling for Path Depth

Also called: Density and Anisotropy Reduction with Ray Depth, Biasing the Sampling

## Overview
Biased Sampling for Path Depth is a variance reduction technique for volumetric path tracing in dense, highly anisotropic participating media (such as clouds) that exponentially decays the medium's extinction coefficient and anisotropy parameter as the ray depth increases [S1, unofficial]. By reducing the frequency of scattering events and the directionality of scattering at deeper bounces, the method significantly shortens path lengths and accelerates convergence, trading a slight, visually negligible bias for massive performance gains [S1, unofficial].

## How it works
The method modifies the medium parameters used during the scattering event selection and phase function evaluation at each bounce of a volumetric path [S1, unofficial]. It operates on the principle that most direct lighting contribution comes from the first and last scattering events, allowing intermediate bounces to be approximated with reduced physical fidelity [S1, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   $\sigma_t$: The base total extinction coefficient of the medium ($m^{-1}$) [S1, unofficial].
    *   $g$: The base anisotropy parameter of the phase function (typically Henyey-Greenstein or Schlick's approximation) [S1, unofficial].
    *   $d$: The current ray depth (bounce count), starting at $d=0$ for the first scattering event [S1, unofficial].
    *   $b$: The bias factor for density reduction, a scalar in $[0, 1]$ [S1, unofficial].
*   **Outputs:**
    *   $\sigma_t'$: The biased total extinction coefficient used for distance sampling [S1, unofficial].
    *   $g'$: The biased anisotropy parameter used for phase function evaluation [S1, unofficial].

### Data Structures
Not given in the sources.

### Parameters
*   **Bias Factor ($b$):** Controls the rate of exponential decay for the extinction coefficient [S1, unofficial].
    *   **Typical Value:** $0.8$ for cloud rendering [S1, unofficial].
    *   **Range:** $[0, 1)$ [S1, unofficial].

### Step-by-Step Method
1.  **Initialize Depth:** Set ray depth $d = 0$ for the first scattering event [S1, unofficial].
2.  **Compute Biased Extinction:** Calculate the modified extinction coefficient $\sigma_t'$ using the bias factor $b$ and current depth $d$:
    $$ \sigma_{t}^{\'} = \sigma_{t} \cdot b^{d} $$
    This reduces the probability of scattering at deeper bounces, leading to larger sampled distances [S1, unofficial].
3.  **Compute Biased Anisotropy:** Calculate the modified anisotropy parameter $g'$ using the base anisotropy $g$ and current depth $d$:
    $$ g^\prime = |g|^{1 + d} \cdot \text{sign}(g) $$
    This reduces the forward-peaked nature of scattering at deeper bounces, making the medium appear more isotropic and reducing the variance associated with highly directional scattering [S1, unofficial].
4.  **Sample Distance:** Use $\sigma_t'$ to sample the distance to the next scattering event (e.g., via inverse CDF sampling of the Beer-Lambert law) [S1, unofficial].
5.  **Sample Phase:** Use $g'$ in the phase function (e.g., Henyey-Greenstein or Schlick's) to sample the outgoing direction [S1, unofficial].
6.  **Increment Depth:** Increment $d$ for the next scattering event [S1, unofficial].
7.  **Optional Optimization:** To further reduce variance and cost, calculate direct lighting (Next Event Estimation) only at the first ($d=0$) and last scattering events of the path [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def sample_biased_medium_step(base_sigma_t, base_g, bias_factor, ray_depth):
    """Computes the biased extinction and anisotropy for a given ray depth."""
    biased_sigma_t = base_sigma_t * (bias_factor ** ray_depth)
    biased_g = (abs(base_g) ** (1 + ray_depth)) * (1 if base_g >= 0 else -1)
    return biased_sigma_t, biased_g

def sample_henyey_greenstein(g, rng_state):
    """Samples cos_theta from the Henyey-Greenstein phase function [G]."""
    if abs(g) < 1e-6:
        cos_theta = 2.0 * rng_state.uniform(0.0, 1.0) - 1.0
    else:
        u = rng_state.uniform(0.0, 1.0)
        cos_theta = (1.0 + g**2 - ((1.0 - g**2) / (1.0 - g + 2.0 * g * u))**2) / (2.0 * g)
        cos_theta = max(-1.0, min(1.0, cos_theta))
    return cos_theta

def rotate_vector(direction, cos_theta, sin_theta, rng_state):
    """Rotates a direction vector by cos_theta and sin_theta around a random azimuth [G]."""
    phi = rng_state.uniform(0.0, 2.0 * math.pi)
    sin_phi = math.sin(phi)
    cos_phi = math.cos(phi)
    # Construct orthonormal basis perpendicular to direction
    if abs(direction[0]) < 0.9:
        perp1 = [1.0, 0.0, 0.0]
    else:
        perp1 = [0.0, 1.0, 0.0]
    perp1 = [perp1[i] - direction[i] * sum(d*p for d, p in zip(direction, perp1)) for i in range(3)]
    perp1_len = math.sqrt(sum(x**2 for x in perp1))
    perp1 = [x / perp1_len for x in perp1]
    perp2 = [direction[1]*perp1[2] - direction[2]*perp1[1],
             direction[2]*perp1[0] - direction[0]*perp1[2],
             direction[0]*perp1[1] - direction[1]*perp1[0]]
    # Rotate
    new_dir = [
        direction[0] * cos_theta + perp1[0] * sin_theta * cos_phi + perp2[0] * sin_theta * sin_phi,
        direction[1] * cos_theta + perp1[1] * sin_theta * cos_phi + perp2[1] * sin_theta * sin_phi,
        direction[2] * cos_theta + perp1[2] * sin_theta * cos_phi + perp2[2] * sin_theta * sin_phi
    ]
    return new_dir

def volumetric_path_step(base_sigma_t, base_g, bias_factor, ray_depth, current_position, ray_direction, rng_state):
    """Performs one step of biased volumetric path tracing."""
    sigma_t_prime, g_prime = sample_biased_medium_step(base_sigma_t, base_g, bias_factor, ray_depth)
    
    if sigma_t_prime <= 0:
        return None, None, 1.0
    
    u = rng_state.uniform(0.0, 1.0)
    distance = -math.log(u) / sigma_t_prime
    
    next_position = [current_position[i] + distance * ray_direction[i] for i in range(3)]
    cos_theta = sample_henyey_greenstein(g_prime, rng_state)
    sin_theta = math.sqrt(max(0.0, 1.0 - cos_theta**2))
    outgoing_direction = rotate_vector(ray_direction, cos_theta, sin_theta, rng_state)
    
    return next_position, outgoing_direction, 1.0
```

### Edge Cases and Pitfalls
*   **Zero Extinction:** If $b=0$ and $d \ge 1$, $\sigma_t'$ becomes $0$ [G]. The distance sampling formula $-\ln(u)/\sigma_t'$ results in division by zero [G]. A guard clause should handle $\sigma_t' \le \epsilon$ by terminating the path or treating it as a vacuum [G].
*   **Bias Introduction:** The method introduces bias [S1, unofficial]. Without biasing, low sample counts exhibit high-frequency noise that makes clouds appear darker than they are [S1, unofficial]. Biased sampling reduces this noise, making the image appear brighter, but this brightness is primarily due to reduced variance, not increased radiance [S1, unofficial].
*   **Visual Difference:** The visual difference between biased and unbiased rendering is minimal in practice, but the bias is present [S1, unofficial].
*   **Reduction Strategy:** The original Hyperion paper suggests linear reduction between the 5th and 20th bounce [S1, unofficial]. However, reducing density and anisotropy throughout the entire path (exponential decay from $d=0$) was found to have minimal visual difference while being faster and resulting in fewer sampled events [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The method reduces the average path length by decreasing the probability of scattering at deeper bounces [S1, unofficial].
*   **Memory Complexity:** No additional memory is required beyond standard path tracing buffers [G].
*   **Measured Performance:** In cloud rendering tests:
    *   Basic Delta Tracking: ~248 seconds for 10 samples per pixel (spp) [S1, unofficial].
    *   With Biased Sampling ($b=0.8$): ~1.1 seconds for 10 spp (225x speedup) [S1, unofficial].
    *   Clean image at 1000 spp: ~123 seconds [S1, unofficial].
    *   High-density clouds at 5000 spp: ~996 seconds [S1, unofficial].
*   **Strengths:** Drastically reduces render times for dense, highly anisotropic media. Reduces variance significantly, allowing for cleaner images at low sample counts [S1, unofficial].
*   **Limitations:** Introduces bias. Not suitable for applications requiring strict energy conservation or unbiased results [S1, unofficial].
*   **When to Use:** Cloud rendering, smoke, fog, and other dense participating media where high anisotropy and density cause excessive scattering events and high variance [S1, unofficial].
*   **When Not to Use:** Unbiased rendering requirements, media with low density or isotropic scattering, or when strict physical accuracy is required for scientific visualization [G].

## Background
The technique is derived from Disney's Hyperion renderer, as described in "The Design and Evolution of Disney’s Hyperion Renderer" [S1, unofficial]. Hyperion is a production-quality renderer used for animated films [G]. The method was implemented to address the computational cost of rendering dense clouds, which involve many scattering events over short distances in similar directions [S1, unofficial]. The author of the source implemented the exponential decay strategy globally rather than the linear reduction between bounces 5 and 20 suggested in the paper, noting minimal visual difference while achieving faster performance due to fewer sampled events [S1, unofficial].

## Sources
[S1] Volumetric Rendering using Path Tracing, https://zydak.github.io/Clouds/index.html, fetched 2026-10-27 (unofficial)
