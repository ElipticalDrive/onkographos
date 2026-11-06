# Biased Sampling for Path Depth

Also called: Density and Anisotropy Reduction with Ray Depth, Biasing the Sampling

## Overview
Biased Sampling for Path Depth is a variance reduction technique for volumetric path tracing in dense, highly anisotropic participating media (such as clouds) that exponentially decays the medium's extinction coefficient and anisotropy parameter as the ray depth increases [S1, unofficial]. By reducing the frequency of scattering events and the directionality of scattering at deeper bounces, the method significantly shortens path lengths and accelerates convergence, trading a slight, visually negligible bias for massive performance gains [S1, unofficial]. It is distinct from unbiased variance reduction methods like [Multiple Importance Sampling for Volumetric Path Tracing](1167-multiple-importance-sampling-for-volumetric-path-tracing.md) or [Delta tracking](985-delta-tracking.md) by explicitly altering the physical properties of the medium along the path rather than just the sampling distribution.

## How it works
The method modifies the medium parameters used during the scattering event selection and phase function evaluation at each bounce of a volumetric path. It operates on the principle that most direct lighting contribution comes from the first and last scattering events, allowing intermediate bounces to be approximated with reduced physical fidelity [S1, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   $\sigma_t$: The base total extinction coefficient of the medium ($m^{-1}$).
    *   $g$: The base anisotropy parameter of the phase function (typically Henyey-Greenstein or Schlick's approximation).
    *   $d$: The current ray depth (bounce count), starting at $d=0$ for the first scattering event.
    *   $b$: The bias factor for density reduction, a scalar in $[0, 1]$.
*   **Outputs:**
    *   $\sigma_t'$: The biased total extinction coefficient used for distance sampling.
    *   $g'$: The biased anisotropy parameter used for phase function evaluation.

### Data Structures
No specific complex data structures are required beyond standard scalar variables for the current bounce depth and the base medium properties. The biasing is applied dynamically during the path traversal loop.

### Parameters
*   **Bias Factor ($b$):** Controls the rate of exponential decay for the extinction coefficient.
    *   **Typical Value:** $0.8$ for cloud rendering [S1, unofficial].
    *   **Range:** $[0, 1)$ [S1, unofficial].

### Step-by-Step Method
1.  **Initialize Depth:** Set ray depth $d = 0$ for the first scattering event.
2.  **Compute Biased Extinction:** Calculate the modified extinction coefficient $\sigma_t'$ using the bias factor $b$ and current depth $d$:
    $$ \sigma_{t}^{\'} = \sigma_{t} \cdot b^{d} $$
    This reduces the probability of scattering at deeper bounces, leading to larger sampled distances [S1, unofficial].
3.  **Compute Biased Anisotropy:** Calculate the modified anisotropy parameter $g'$ using the base anisotropy $g$ and current depth $d$:
    $$ g^\prime = |g|^{1 + d} \cdot \text{sign}(g) $$
    This reduces the forward-peaked nature of scattering at deeper bounces, making the medium appear more isotropic and reducing the variance associated with highly directional scattering [S1, unofficial].
4.  **Sample Distance:** Use $\sigma_t'$ to sample the distance to the next scattering event (e.g., via inverse CDF sampling of the Beer-Lambert law) [S1, unofficial].
5.  **Sample Phase:** Use $g'$ in the phase function (e.g., Henyey-Greenstein or Schlick's) to sample the outgoing direction [S1, unofficial].
6.  **Increment Depth:** Increment $d$ for the next scattering event.
7.  **Optional Optimization:** To further reduce variance and cost, calculate direct lighting (Next Event Estimation) only at the first ($d=0$) and last scattering events of the path. Intermediate bounces skip direct lighting calculations [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_biased_medium_step(base_sigma_t, base_g, bias_factor, ray_depth):
    """
    Computes the biased extinction and anisotropy for a given ray depth.
    
    Args:
        base_sigma_t (float): The original total extinction coefficient.
        base_g (float): The original anisotropy parameter.
        bias_factor (float): The density decay factor b (0 <= b < 1).
        ray_depth (int): The current bounce count d.
        
    Returns:
        tuple: (biased_sigma_t, biased_g)
    """
    # Calculate biased extinction coefficient
    # sigma_t' = sigma_t * b^d
    biased_sigma_t = base_sigma_t * (bias_factor ** ray_depth)
    
    # Calculate biased anisotropy parameter
    # g' = |g|^(1+d) * sign(g)
    biased_g = (abs(base_g) ** (1 + ray_depth)) * (1 if base_g >= 0 else -1)
    
    return biased_sigma_t, biased_g

def volumetric_path_step(base_sigma_t, base_g, bias_factor, ray_depth, rng):
    """
    Performs one step of biased volumetric path tracing.
    
    Args:
        base_sigma_t (float): The original total extinction coefficient.
        base_g (float): The original anisotropy parameter.
        bias_factor (float): The density decay factor b.
        ray_depth (int): The current bounce count d.
        rng (RandomNumberGenerator): Random number generator instance.
        
    Returns:
        tuple: (next_position, outgoing_direction, transmittance_weight)
    """
    # 1. Get biased parameters
    sigma_t_prime, g_prime = sample_biased_medium_step(base_sigma_t, base_g, bias_factor, ray_depth)
    
    # 2. Sample distance to next scattering event
    # Using inverse CDF for homogeneous medium: t = -ln(u) / sigma_t'
    # If sigma_t_prime is 0 (e.g., at very high depth with b=0), sample infinity or terminate
    if sigma_t_prime <= 0:
        # No scattering occurs; ray exits medium or terminates
        return None, None, 1.0
    
    u = rng.uniform(0.0, 1.0)
    distance = -math.log(u) / sigma_t_prime
    
    # 3. Move to next position
    next_position = current_position + distance * ray_direction
    
    # 4. Sample outgoing direction using biased anisotropy
    # Assuming Henyey-Greenstein phase function for sampling
    # cos_theta = ((1 - g'^2) / (1 - g' + 2*g'*u)) if g' != 0 else 0.5
    # For g' = 0, isotropic scattering: cos_theta = 2*u - 1
    if abs(g_prime) < 1e-6:
        cos_theta = 2.0 * rng.uniform(0.0, 1.0) - 1.0
    else:
        u2 = rng.uniform(0.0, 1.0)
        cos_theta = (1.0 + g_prime**2 - ((1.0 - g_prime**2) / (1.0 - g_prime + 2.0 * g_prime * u2)))
        # Clamp to [-1, 1] for numerical stability
        cos_theta = max(-1.0, min(1.0, cos_theta))
    
    # Generate outgoing direction from cos_theta and random azimuth
    # (Implementation details for vector rotation omitted as per standard graphics practice)
    outgoing_direction = sample_henyey_greenstein_direction(ray_direction, cos_theta, rng)
    
    # 5. Calculate transmittance weight for the path segment
    # T = exp(-sigma_t * distance) using the BASE sigma_t for physical correctness of energy conservation
    # Note: The biased sigma_t' is only for sampling distance, not for transmittance calculation
    transmittance = math.exp(-base_sigma_t * distance)
    
    return next_position, outgoing_direction, transmittance
```

### Edge Cases and Pitfalls
*   **Zero Extinction:** If $b=0$ and $d \ge 1$, $\sigma_t'$ becomes 0. The distance sampling formula $-\ln(u)/\sigma_t'$ results in division by zero. In practice, this implies the ray travels infinitely far or terminates. A guard clause should handle $\sigma_t' \le \epsilon$ by terminating the path or treating it as a vacuum [G].
*   **Bias Introduction:** The method introduces bias. Without biasing, low sample counts exhibit high-frequency noise that makes clouds appear darker than they are. Biased sampling reduces this noise, making the image appear brighter, but this brightness is primarily due to reduced variance, not increased radiance [S1, unofficial].
*   **Visual Difference:** The visual difference between biased and unbiased rendering is minimal in practice, but the bias is present [S1, unofficial].
*   **Reduction Strategy:** The original Hyperion paper suggests linear reduction between the 5th and 20th bounce. However, reducing density and anisotropy throughout the entire path (exponential decay from $d=0$) was found to have no significant visual difference while being faster and resulting in fewer sampled events [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The method reduces the average path length by decreasing the probability of scattering at deeper bounces. This leads to a significant reduction in the number of scattering events per path.
*   **Memory Complexity:** No additional memory is required beyond standard path tracing buffers.
*   **Measured Performance:** In cloud rendering tests:
    *   Basic Delta Tracking: ~248 seconds for 10 spp [S1, unofficial].
    *   With Biased Sampling ($b=0.8$): ~1.1 seconds for 10 spp (225x speedup) [S1, unofficial].
    *   Clean image at 1000 spp: ~123 seconds [S1, unofficial].
    *   High-density clouds at 5000 spp: ~996 seconds [S1, unofficial].
*   **Strengths:** Drastically reduces render times for dense, highly anisotropic media. Reduces variance significantly, allowing for cleaner images at low sample counts.
*   **Limitations:** Introduces bias. Not suitable for applications requiring strict energy conservation or unbiased results. Performance gains diminish if the medium is not dense or anisotropic.
*   **When to Use:** Cloud rendering, smoke, fog, and other dense participating media where high anisotropy and density cause excessive scattering events and high variance.
*   **When Not to Use:** Unbiased rendering requirements, media with low density or isotropic scattering, or when strict physical accuracy is required for scientific visualization.

## Background
The technique is derived from Disney's Hyperion renderer, as described in "The Design and Evolution of Disney’s Hyperion Renderer" [S1, unofficial]. Hyperion is a production-quality renderer used for animated films. The method was implemented to address the computational cost of rendering dense clouds, which involve many scattering events over short distances in similar directions. The author of the source implemented the exponential decay strategy globally rather than the linear reduction between bounces 5 and 20 suggested in the paper, finding no visual difference but achieving better performance [S1, unofficial].

## Sources
[S1] Volumetric Rendering using Path Tracing, https://zydak.github.io/Clouds/index.html, fetched 2026-10-27 (unofficial)
