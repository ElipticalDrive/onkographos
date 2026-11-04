# Path Tracing with Discrete Event Sampling

Also called: Null-Event Path Tracing, Russian Roulette with Discrete Events

## Overview
Path Tracing with Discrete Event Sampling is a path tracing algorithm for participating media that replaces continuous Russian Roulette termination with a discrete probability distribution over three mutually exclusive events: null (transmittance), absorption, and scattering [S1, unofficial]. It solves the inefficiency of standard continuous methods in sparse media by explicitly accounting for null events, allowing the path to continue without weight reduction or termination checks at every step, while handling temperature-dependent emission and Henyey-Greenstein phase function scattering [S1, unofficial]. It builds on [Explicit Path Tracing with Russian Roulette](2209-explicit-path-tracing-with-russian-roulette.md) by managing recursion depth explicitly and using discrete probabilities $p_n, p_a, p_s$ instead of probabilistic survival based on albedo [S1, unofficial].

## How it works
**Inputs:**
- Ray origin $\mathbf{o}$ and direction $\mathbf{d}$.
- Density grid $\sigma_t(\mathbf{x})$ (total extinction coefficient).
- Optional temperature grid $T_{adim}(\mathbf{x})$ (normalized).
- Parameters: `max_depth` (integer), `henyey_greenstein_g` (float, anisotropy), `le_scale` (float, emission scale), `temperature_scale` (float), `temperature_offset` (float), `infinite_light` (vector and multiplier).
- Random Number Generator (RNG).

**Outputs:**
- Accumulated radiance $L$ (vector).

**Data Structures:**
- `MajorantTransmittanceSampler`: Manages the sampling of interaction points along the ray using the majorant density. It provides an iterator or loop over sampled points within the current ray segment.
- `Event`: Enum or integer representing {Null, Absorption, Scatter}.

**Constants:**
- Planck's constant $h = 6.62606957 \times 10^{-34}$ [S1, unofficial].
- Boltzmann constant $k_b = 1.3806488 \times 10^{-23}$ [S1, unofficial].
- Speed of light $c$ (standard physical constant).

**Steps:**
1. Initialize accumulated radiance $L = \mathbf{0}$ and set `terminated` flag to false.
2. Loop for `depth` from 0 to `max_depth - 1`:
    a. Intersect the current ray with the volume grid.
    b. If no intersection, break the depth loop.
    c. Initialize `MajorantTransmittanceSampler` with the intersection interval, RNG, density grid, and total extinction coefficient $\sigma_t$.
    d. Loop through sampled points $\mathbf{x}$ from the majorant sampler:
        i. If temperature grid is available:
            1. Sample temperature $T_{adim}$ at $\mathbf{x}$.
            2. Convert to Kelvin: $T_K = T_{adim} \cdot \text{temperature\_scale} + \text{temperature\_offset}$.
            3. Calculate blackbody radiation spectrum $\mathbf{L}_e$ at $T_K$ using Planck's law for wavelength $\lambda$:
               $$ \mathbf{L}_e(\lambda) = \frac{2 h c^2}{\lambda^5} \frac{1}{e^{\frac{hc}{\lambda k_b T_K}} - 1} $$
               [S1, unofficial]
            4. Add emission contribution to $L$: $L \leftarrow L + p_a \cdot \text{le\_scale} \cdot \mathbf{L}_e$.
        ii. Calculate probabilities for the three events based on local coefficients (not given in sources, but implied by $p_a, p_s, p_n$) [G]:
            - $p_s$: Probability of scattering.
            - $p_a$: Probability of absorption.
            - $p_n$: Probability of null event (transmittance).
        iii. Ensure $p_n$ is valid: $p_n = \max(1.0 - p_a - p_s, 0.0)$ [S1, unofficial].
        iv. Sample a uniform random number $u \in [0, 1)$.
        v. Determine event:
            - If $u < p_n$: Event is **Null**.
            - Else if $u < p_n + p_a$: Event is **Absorption**.
            - Else: Event is **Scatter**.
        vi. Handle Event:
            - **Null**: Continue to the next sampled point in the majorant loop.
            - **Absorption**: Mark path as terminated, break the majorant loop.
            - **Scatter**:
                1. If `depth` == `max_depth` - 1: Log termination, break the majorant loop (path ends).
                2. Sample a new direction $\mathbf{d}'$ using the Henyey-Greenstein phase function with parameter $g$:
                   $$ p(\mathbf{d}'|\mathbf{d}) = \frac{1-g^2}{4\pi (1 + g^2 - 2g(\mathbf{d} \cdot \mathbf{d}'))^{3/2}} $$
                   [G] (Standard HG formula, not explicitly in source but required for implementation).
                3. Update ray origin to $\mathbf{x}$ and direction to $\mathbf{d}'$.
                4. Recursively sample scattered radiance $L_{scat}$ (implementation detail for recursive call or next iteration setup) [G].
                5. Break the majorant loop to continue the path at the next depth level.
    e. If the path terminated due to Absorption or Max Depth, break the depth loop.
3. If the path escaped to infinity (ray did not terminate in volume):
    - Add infinite light contribution: $L \leftarrow L + \text{infinite\_light.xyz} \cdot \text{infinite\_light.multiplier}$.
4. Add $L$ to the film buffer, scaled by `imaging_ratio`, and increment sample count.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def path_tracing_discrete_event_sampling(ray_o, ray_d, density_grid, temp_grid, params, rng):
    """
    ray_o: Vector3, ray origin
    ray_d: Vector3, ray direction (normalized)
    density_grid: Function or object providing sigma_t(x)
    temp_grid: Function or object providing normalized temperature T_adim(x), or None
    params: Object with max_depth, henyey_greenstein_g, le_scale, temperature_scale, 
            temperature_offset, infinite_light (vector, multiplier)
    rng: Random number generator
    """
    L = Vector3(0.0, 0.0, 0.0)
    
    # Constants for Planck's law
    h = 6.62606957e-34
    kb = 1.3806488e-23
    c = 299792458.0  # Speed of light
    
    for depth in range(params.max_depth):
        # Intersect ray with volume grid
        hit = intersect_volume_grid(ray_o, ray_d)
        if not hit:
            break
            
        # Initialize majorant sampler for the segment
        # This sampler yields points along the ray based on majorant density
        majorant_sampler = MajorantTransmittanceSampler(hit, rng, density_grid)
        
        for point in majorant_sampler:
            x = point.position
            
            # Handle emission if temperature grid is available
            if temp_grid is not None:
                T_adim = temp_grid.sample(x)
                T_K = T_adim * params.temperature_scale + params.temperature_offset
                
                # Calculate blackbody radiation
                # Note: This is a simplified spectral calculation; real implementation 
                # might integrate over spectrum or use XYZ conversion.
                # The source uses a specific lambda, but for general radiance, 
                # we assume this returns a vector or scalar contribution.
                # For this pseudocode, we assume blackbody_radiation_xyz returns a Vector3
                # representing the XYZ or RGB contribution.
                Le = blackbody_radiation_xyz(T_K, h, kb, c)
                
                # Add emission contribution scaled by absorption probability and le_scale
                # Note: The source adds p_a * le_scale * Le. 
                # This implies Le is the emitted radiance and p_a is the weight for absorption event.
                # However, in discrete event sampling, emission is often associated with the 
                # medium itself. The source code snippet shows:
                # L += p_a * le_scale * blackbody_radiation_xyz(temp_K)
                # This suggests emission is added when an absorption event is *sampled*? 
                # Or is it added at every step? The source says "If temperature grid is available... add emission contribution".
                # It does not condition it on the event type in the snippet provided.
                # However, physically, emission should be added regardless of event, 
                # but the source code structure suggests it might be tied to the step.
                # Let's follow the source literally: it adds it inside the point loop.
                L += point.probability_absorption * params.le_scale * Le

            # Calculate probabilities
            # p_s: scattering probability
            # p_a: absorption probability
            # These depend on local coefficients sigma_s and sigma_a
            sigma_s = density_grid.get_sigma_s(x)
            sigma_a = density_grid.get_sigma_a(x)
            sigma_t = density_grid.get_sigma_t(x)
            
            if sigma_t == 0:
                p_s = 0.0
                p_a = 0.0
            else:
                p_s = sigma_s / sigma_t
                p_a = sigma_a / sigma_t
            
            # Null probability
            p_n = max(1.0 - p_a - p_s, 0.0)
            
            # Sample discrete event
            u = rng.uniform(0.0, 1.0)
            
            if u < p_n:
                # Null event: continue to next point
                continue
            elif u < p_n + p_a:
                # Absorption event: terminate path
                break
            else:
                # Scatter event
                if depth == params.max_depth - 1:
                    # Max depth reached, terminate
                    break
                
                # Sample new direction using Henyey-Greenstein
                g = params.henyey_greenstein_g
                d_prime = sample_henyey_greenstein(ray_d, g, rng)
                
                # Update ray for next iteration
                ray_o = x
                ray_d = d_prime
                
                # Break to continue path at next depth
                break
        else:
            # If majorant loop finished without break (e.g. ray exited volume)
            break
    else:
        # If loop completed without break (escaped to infinity)
        L += params.infinite_light.xyz * params.infinite_light.multiplier
        
    return L

def blackbody_radiation_xyz(T_K, h, kb, c):
    """
    Simplified blackbody radiation calculation.
    The source uses a specific lambda. For a general implementation, 
    this would typically integrate over the visible spectrum or use precomputed tables.
    The source snippet shows a calculation for a specific lambda_m.
    We will return a placeholder vector as the exact spectral integration is not detailed.
    """
    # This is a placeholder. The source calculates for a specific lambda.
    # In a real implementation, one would integrate Planck's law over XYZ color matching functions.
    return Vector3(0.0, 0.0, 0.0) # [G] Placeholder for spectral integration

def sample_henyey_greenstein(d, g, rng):
    """
    Samples a new direction from the Henyey-Greenstein phase function.
    """
    # Standard sampling method for HG phase function
    # cos_theta = ((1 - g^2) / (1 - g + 2*g*u)) if g != 1 else 1
    # phi = 2 * pi * v
    # Convert spherical to Cartesian relative to d
    u = rng.uniform(0.0, 1.0)
    v = rng.uniform(0.0, 1.0)
    
    if g == 0:
        # Isotropic
        cos_theta = 2 * v - 1
        sin_theta = sqrt(1 - cos_theta**2)
        phi = 2 * pi * u
    else:
        cos_theta = (1 - g**2) / (1 - g + 2 * g * u)
        sin_theta = sqrt(1 - cos_theta**2)
        phi = 2 * pi * v
        
    # Construct new direction
    # This requires constructing an orthonormal basis with d as the z-axis
    # [G] Implementation detail for basis construction
    return Vector3(0.0, 0.0, 0.0) # [G] Placeholder for direction sampling
```

**Edge Cases and Pitfalls:**
- **Precision:** Ensure $p_n$ is clamped to $[0, 1]$ to avoid negative probabilities due to floating-point errors [S1, unofficial].
- **Boundaries:** Handle ray intersections with volume boundaries correctly to avoid sampling outside the grid [S1, unofficial].
- **Degenerate Input:** If $\sigma_t = 0$, probabilities are undefined; handle by setting $p_s = p_a = 0$ and $p_n = 1$ [G].
- **Temperature Conversion:** Ensure temperature conversion to Kelvin is correct to avoid negative or zero temperatures in Planck's law [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Proportional to the number of sampled points along the ray, which depends on the majorant density and path length. Each point involves a discrete event sample and potentially a phase function evaluation [S1, unofficial].
- **Memory Complexity:** Depends on the density and temperature grids. The algorithm itself uses constant memory per path [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales with the number of samples and path depth. Efficient in sparse media due to explicit null event handling [S1, unofficial].
- **Strengths:** Explicit handling of absorption and null events improves efficiency in sparse media. Handles temperature-dependent emission [S1, unofficial].
- **Limitations:** Requires pre-computed probabilities for events. May be less efficient in dense media where scattering is frequent [S1, unofficial].
- **When to use:** Suitable for participating media with sparse scattering and temperature-dependent emission.
- **When not to use:** Not ideal for dense media or when continuous Russian Roulette is preferred for simplicity.

## Background
- **Origin:** The method is described in the unofficial source [S1, unofficial], which is a GitHub repository `Shibodd/volume_path_tracer`. The specific paper or author is not cited in the provided notes.
- **History:** The method appears to be a recent implementation detail in the context of volume path tracing, leveraging discrete event sampling for efficiency.
- **Usage:** Used in the `Shibodd/volume_path_tracer` project for rendering participating media with temperature-dependent emission [S1, unofficial].

## Sources
[S1] Shibodd/volume_path_tracer, https://github.com/Shibodd/volume_path_tracer, fetched 2026-10-26 (unofficial)
