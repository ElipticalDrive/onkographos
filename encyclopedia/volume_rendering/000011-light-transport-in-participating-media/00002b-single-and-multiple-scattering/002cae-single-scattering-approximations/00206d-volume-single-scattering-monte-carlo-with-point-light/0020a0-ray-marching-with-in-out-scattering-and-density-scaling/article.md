# Ray Marching with In/Out-Scattering and Density Scaling

Also called: Out-scattering, Out-scattering Model

## Overview
This method computes single scattering radiance in participating media by ray marching, explicitly accounting for four light-particle interactions: absorption, out-scattering (attenuation), emission (ignored), and in-scattering (energy gain) [S2, unofficial]. It distinguishes between homogeneous media (uniform coefficients) and heterogeneous media (spatially varying density) by using a density scalar to scale extinction and scattering terms [S2, unofficial]. Unlike the parent method [Volume Single Scattering (Monte Carlo with Point Light)](2293-volume-single-scattering-monte-carlo-with-point-light.md) which uses Monte Carlo integration, this approach uses deterministic uniform sampling along the ray, and unlike [Single Scattering Approximation with Precomputed Terms](4188-single-scattering-approximation-with-precomputed-terms.md) or [Analytical Approximation of Single Scattering Radiance](4244-analytical-approximation-of-single-scattering-radiance.md), it does not rely on precomputed terms or closed-form solutions but iteratively accumulates light contributions per step [S2, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- Ray origin $O$ and direction $D$.
- Medium definition: absorption coefficient $\sigma_a$, scattering coefficient $\sigma_s$, and density function $\rho(x)$ [S2, unofficial].
- Light source position $L_{pos}$ and color $C_{light}$ [S2, unofficial].
- Background color $C_{bg}$ [S2, unofficial].
- Step size $h$ (distance traveled per ray march step) [S2, unofficial].
- Phase function $p(x, \omega, \omega')$ [S2, unofficial].

**Outputs:**
- Final radiance $L$ at the ray origin [S2, unofficial].

### Data Structures
- **Sample Point**: Position $x$ along the ray.
- **Intersection Volume**: Structure containing `inside` (boolean) and `t1` (float, distance from sample to light volume boundary along light direction) [S2, unofficial].
- **Accumulators**:
  - `transparency`: Float, initialized to 1.0. Represents the accumulated transmission from the eye to the current sample [S2, unofficial].
  - `result`: Float (or vector), initialized to 0.0. Accumulates in-scattered radiance [S2, unofficial].

### Formulas
1.  **Extinction Coefficient**:
    $$ \sigma_t(x) = \sigma_a(x) + \sigma_s(x) $$
    Where $\sigma_a$ is the absorption coefficient and $\sigma_s$ is the scattering coefficient [S1, S2, unofficial]. Note: The source text contains a typo $\sigma_t = \sigma_a + \sigma_t$, but the correct physical definition and code usage confirm $\sigma_t = \sigma_a + \sigma_s$ [S2, unofficial].

2.  **Out-Scattering (Attenuation)**:
    The differential change in incident radiance due to out-scattering is:
    $$ \mathrm{d} L_{\mathrm{i}} (x, \omega) := -\sigma_{\mathrm{s}} (x, \omega) L_{\mathrm{i}} (x, \omega) \; \mathrm{d}t $$
    [S1, unofficial]
    The total attenuation (extinction) over a step $h$ is computed using Beer's Law:
    $$ T_{step} = e^{-\rho(x) \cdot h \cdot (\sigma_a + \sigma_s)} $$
    [S2, unofficial]

3.  **In-Scattering**:
    The in-scattering radiance equation is:
    $$ L_i(x, \omega) = \sigma_s \int_{S^2} p(x, \omega, \omega') L(x, \omega') d\omega' $$
    [S2, unofficial]
    For a single point light, the incident radiance $L(x, \omega')$ is attenuated by the path from the light to the sample. The light attenuation factor is:
    $$ A_{light} = e^{-\rho(x) \cdot t_1 \cdot (\sigma_a + \sigma_s)} $$
    Where $t_1$ is the distance the light travels through the volumetric sphere to the sample [S2, unofficial].

4.  **Phase Function**:
    $p(x, \omega, \omega')$ models the angular distribution of scattered light. It returns a value in $[0, 1]$ and integrates to 1 over the sphere of directions $S^2$ [S2, unofficial].

### Parameters
- $\sigma_a$: Absorption coefficient. Typical value: 0.5 [S2, unofficial].
- $\sigma_s$: Scattering coefficient. Typical value: 0.5 [S2, unofficial].
- $\rho(x)$: Density. Scales $\sigma_a$ and $\sigma_s$ globally. Typical value: 1.0 for homogeneous media [S2, unofficial].
- $h$: Step size. Determines the resolution of the ray march [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march_in_out_scattering(ray_origin, ray_direction, light_pos, light_color, background_color, sigma_a, sigma_s, density_func, step_size, phase_func):
    """
    Computes single scattering radiance using ray marching with in/out-scattering and density scaling.
    
    Args:
        ray_origin: 3D vector, origin of the camera ray.
        ray_direction: 3D vector, normalized direction of the camera ray.
        light_pos: 3D vector, position of the point light source.
        light_color: 3D vector, color/intensity of the light.
        background_color: 3D vector, color of the background.
        sigma_a: float, absorption coefficient.
        sigma_s: float, scattering coefficient.
        density_func: function(x) -> float, returns density at position x.
        step_size: float, distance per ray march step.
        phase_func: function(view_dir, light_dir) -> float, returns phase function value.
        
    Returns:
        3D vector, final radiance.
    """
    # Initialize accumulators
    transparency = 1.0
    result = 0.0
    
    # Current position along the ray
    current_pos = ray_origin
    
    # Determine ray length (simplified: fixed length or until transparency is negligible)
    # In a full implementation, this would involve ray-box or ray-sphere intersection
    # to find the entry and exit points of the volume.
    # Here we assume a loop over steps within the volume.
    
    max_steps = 100 # Placeholder for actual volume boundary logic
    for i in range(max_steps):
        # 1. Compute sample transmission (Out-scattering/Attenuation)
        # Calculate extinction coefficient
        sigma_t = sigma_a + sigma_s
        
        # Get density at current position
        rho = density_func(current_pos)
        
        # Compute attenuation for this step
        # T = exp(-density * step_size * (sigma_a + sigma_s))
        sample_attenuation = exp(-rho * step_size * sigma_t)
        
        # Update transparency
        transparency *= sample_attenuation
        
        # 2. Calculate In-Scattering Contribution
        # Vector from sample to light
        light_dir = normalize(light_pos - current_pos)
        
        # Check if the sample is inside the light volume (e.g., a sphere around the light)
        # This is a simplified check. In practice, intersect the ray from current_pos to light_pos
        # with the light volume boundary to find t1.
        isect_vol = intersect_volume(current_pos, light_dir)
        
        if isect_vol.inside:
            t1 = isect_vol.t1 # Distance from sample to light volume boundary
            
            # Light attenuation from light source to sample
            # A_light = exp(-density * t1 * (sigma_a + sigma_s))
            light_attenuation = exp(-rho * t1 * sigma_t)
            
            # Compute phase function value
            # Note: phase_func expects view_dir and light_dir.
            # View direction is ray_direction (from eye to scene).
            # Light direction is from scene to light.
            p_val = phase_func(ray_direction, light_dir)
            
            # Accumulate in-scattering
            # result += transparency * light_color * light_attenuation * sigma_s * density * step_size
            # Note: density is rho.
            in_scatter_contribution = transparency * light_color * light_attenuation * sigma_s * rho * step_size * p_val
            result += in_scatter_contribution
            
        # 3. Advance ray
        current_pos += ray_direction * step_size
        
        # 4. Early exit if transparency is very low
        if transparency < 1e-4:
            break
            
    # 5. Composite final color
    # result is pre-multiplied by density (via rho in the accumulation step),
    # so it is 0 where density is 0.
    final_color = background_color * transparency + result
    
    return final_color

def intersect_volume(origin, direction):
    """
    Simplified intersection with a spherical light volume.
    Returns a structure with 'inside' (bool) and 't1' (float).
    """
    # Placeholder for actual geometric intersection logic
    # This would typically involve solving a quadratic equation for sphere intersection
    # and returning the distance to the boundary along the light direction.
    # For the purpose of this method description, we assume this helper exists.
    pass
```

### Edge Cases and Pitfalls
- **Zero Density**: If density $\rho(x) = 0$, the result must be 0. If the density multiplier is omitted in the in-scattering accumulation, non-zero light would be added to empty space, causing visual artifacts [S2, unofficial]. The pseudocode ensures this by multiplying by `rho` in the accumulation step.
- **Phase Function Integration**: The phase function must integrate to 1 over the sphere $S^2$. Failure to do so will result in incorrect energy conservation [S2, unofficial].
- **Light Volume Intersection**: The distance $t_1$ must be correctly calculated as the distance from the sample to the boundary of the light volume along the light direction. Incorrect calculation leads to wrong attenuation [S2, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N)$ where $N$ is the number of ray march steps. Each step involves exponential calculations and vector operations [S2, unofficial].
- **Memory Complexity**: $O(1)$ auxiliary space, as it only stores accumulators and current position [S2, unofficial].
- **Strengths**: Simple to implement; handles heterogeneous media via density function; explicitly models out-scattering for accurate attenuation [S2, unofficial].
- **Limitations**: Uniform step size may be inefficient for sparse volumes; single scattering approximation ignores multiple scattering events; performance degrades with high density or small step sizes [S2, unofficial].
- **When to use**: Real-time rendering or offline rendering where single scattering is sufficient and computational resources are limited compared to full path tracing [S2, unofficial].
- **When not to use**: Scenes requiring multiple scattering effects or very high accuracy in dense media [S2, unofficial].

## Background
The concept of out-scattering and in-scattering is fundamental to volume rendering theory. Out-scattering refers to the reduction in radiance due to light being deflected away from the view direction [S1, unofficial]. The specific ray marching algorithm with density scaling is detailed in Scratchapixel's "Ray Marching: Getting it Right!" lesson [S2, unofficial]. The theoretical background on scattering coefficients and phase functions is attributed to loannis (Yannis) Gkioulekas [S1, unofficial].

## Sources
[S1] [Graphics] Volume Rendering - GitHub Pages, https://leeyngdo.github.io/blog/computer-graphics/2024-03-28-volume-rendering/, fetched 2026-10-27 (unofficial)
[S2] Ray Marching: Getting it Right!, https://www.scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers//ray-marching-get-it-right.html, fetched 2026-10-28 (unofficial)
