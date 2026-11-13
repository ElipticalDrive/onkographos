# Main Ray Marching with Frostbite Integration

Also called: Frostbite Volume Integration

## Overview
Main Ray Marching with Frostbite Integration is a deterministic ray marching technique for computing single scattering in participating media, specifically designed to handle transmittance, inscatter, and early termination efficiently [S1, unofficial]. It differs from its parent method, [Volume Single Scattering (Monte Carlo with Sphere Light)](4549-volume-single-scattering-monte-carlo-with-sphere-light.md), by replacing stochastic sampling with a fixed-step deterministic integration along the view ray, using a specific analytical approximation for the radiance accumulation known as the "Frostbite method" [S1, unofficial].

## How it works
**Inputs:**
- Ray origin $O$ and direction $D$ (normalized).
- Bounding volume of the participating medium (to determine entry/exit points).
- Volume density field $\rho(\mathbf{p})$ (assumed constant or sampled per step).
- Scattering coefficient $\sigma_s$ and extinction coefficient $\sigma_t$ (material properties).
- Step size $s$ (distance between samples).
- Light source parameters (position, intensity, phase function parameters).

**Outputs:**
- Accumulated radiance $L$ at the pixel.
- Total transmittance $T_{total}$ (passed by reference, initialized to 1.0).

**Data Structures:**
- Point $P$: Current position on the ray.
- Vector $D$: Ray direction.
- Scalar $T_{total}$: Cumulative transmittance from origin to current point.
- Scalar $L$: Accumulated radiance.

**Step-by-Step Method:**

1.  **Intersection Test:** Determine if the ray intersects the bounding volume of the volume. If no intersection, return $L=0$ and $T_{total}=0$ [S1, unofficial].
2.  **Initialization:** Set $L = 0$. Initialize $T_{total} = 1.0$ (or pass in initial transmittance). Calculate the phase function value $p(\theta)$ for the current step. The phase function is a mixture of two Henyey-Greenstein distributions:
    $$ p(\theta) = 0.7 \cdot HG(\theta, g=-0.3) + 0.3 \cdot HG(\theta, g=0.3) $$
    where $\theta$ is the angle between the incoming light direction and the view direction [S1, unofficial].
3.  **Loop:** While $T_{total} > \epsilon$ (where $\epsilon = 10^{-3}$) and the ray is within the volume:
    a.  **Sample Coefficients:** If the local density $\rho > 0$:
        $$ \sigma_s' = \sigma_s \cdot \rho $$
        $$ \sigma_t' = \sigma_t \cdot \rho $$
        If $\rho = 0$, set $\sigma_s' = 0$ and $\sigma_t' = 0$ [S1, unofficial].
    b.  **Calculate Insatter ($L_{inscatter}$):**
        $$ L_{inscatter} = L_{emission} + L_{ambient} + L_{sun} \cdot p(\theta) \cdot \tau_{light} $$
        where $L_{emission}$ is distance glow, $L_{ambient}$ is ambient light, $L_{sun}$ is direct sunlight, and $\tau_{light}$ is the transmittance from the light source to the current point $P$ [S1, unofficial].
    c.  **Scale Insatter:**
        $$ L_{inscatter} = L_{inscatter} \cdot \sigma_s' $$
    d.  **Calculate Step Transmittance ($T_{step}$):**
        $$ T_{step} = \exp(-\sigma_t' \cdot s) $$
    e.  **Integrate Radiance (Frostbite Method):**
        $$ L = L + T_{total} \cdot \frac{L_{inscatter} \cdot (1 - T_{step})}{\sigma_t'} $$
        Note: If $\sigma_t' \approx 0$, this term is handled as $L_{inscatter} \cdot s$ to avoid division by zero [G].
    f.  **Update Total Transmittance:**
        $$ T_{total} = T_{total} \cdot T_{step} $$
    g.  **Early Termination Check:** If $|T_{total}| \le 10^{-3}$, set $T_{total} = 0$ and break the loop [S1, unofficial].
    h.  **Step Forward:**
        $$ P = P + D \cdot s $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def frostbite_volume_integration(ray_origin, ray_dir, bounding_volume, sigma_s, sigma_t, step_size, light_source):
    # 1. Intersection Test
    t_min, t_max = intersect_bounding_volume(ray_origin, ray_dir, bounding_volume)
    if t_min is None:
        return 0.0, 0.0

    # Initialize
    L = 0.0
    T_total = 1.0
    epsilon = 1e-3
    
    # Start at entry point
    current_pos = ray_origin + ray_dir * t_min
    current_dist = t_min
    
    # Loop until exit or early termination
    while current_dist < t_max and T_total > epsilon:
        # 3a. Sample Coefficients (assuming uniform density for simplicity, or sample from volume)
        density = 1.0 # Placeholder for density sampling
        sigma_s_prime = sigma_s * density
        sigma_t_prime = sigma_t * density
        
        # 3b. Calculate Insatter
        # Calculate phase function
        # Note: cos_theta depends on light direction and view direction
        cos_theta = dot(normalize(light_source.position - current_pos), -ray_dir)
        p_theta = 0.7 * heney_greenstein(cos_theta, -0.3) + 0.3 * heney_greenstein(cos_theta, 0.3)
        
        # Light transmittance from light to current point
        light_dist = length(light_source.position - current_pos)
        # Assuming vacuum or simple transmittance for light path
        tau_light = exp(-sigma_t_prime * light_dist) 
        
        L_emission = 0.0 # Distance glow placeholder
        L_ambient = 0.0  # Ambient light placeholder
        L_sun = light_source.intensity
        
        L_inscatter = L_emission + L_ambient + L_sun * p_theta * tau_light
        
        # 3c. Scale Insatter
        L_inscatter *= sigma_s_prime
        
        # 3d. Calculate Step Transmittance
        T_step = exp(-sigma_t_prime * step_size)
        
        # 3e. Integrate Radiance (Frostbite Method)
        if sigma_t_prime > 1e-6:
            term = L_inscatter * (1.0 - T_step) / sigma_t_prime
        else:
            term = L_inscatter * step_size
            
        L += T_total * term
        
        # 3f. Update Total Transmittance
        T_total *= T_step
        
        # 3g. Early Termination
        if abs(T_total) <= epsilon:
            T_total = 0.0
            break
            
        # 3h. Step Forward
        current_pos += ray_dir * step_size
        current_dist += step_size
        
    return L, T_total

def heney_greenstein(cos_theta, g):
    # Henyey-Greenstein phase function
    # p(g, theta) = (1 - g^2) / (4 * pi * (1 + g^2 - 2*g*cos_theta)^(3/2))
    numerator = 1.0 - g * g
    denominator = 4.0 * 3.141592653589793 * pow(1.0 + g * g - 2.0 * g * cos_theta, 1.5)
    if denominator == 0:
        return 0.0
    return numerator / denominator
```

**Edge Cases and Pitfalls:**
- **Division by Zero:** If $\sigma_t' = 0$, the term $\frac{1 - T_{step}}{\sigma_t'}$ becomes $0/0$. The limit is $s$, so use $L_{inscatter} \cdot s$ [G].
- **Precision:** $T_{total}$ can underflow to zero quickly in dense volumes. Early termination at $10^{-3}$ is a heuristic to save computation [S1, unofficial].
- **Phase Function:** The specific mix (0.7/-0.3, 0.3/0.3) is hardcoded in the source [S1, unofficial].

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of steps along the ray. $N = \frac{\text{length of ray in volume}}{\text{step size}}$.
- **Memory Complexity:** $O(1)$ per ray, excluding volume data storage.
- **Performance:** Deterministic and fast per pixel, but requires many steps for accuracy in dense media. Early termination helps in transparent regions.
- **Strengths:** Simple to implement, no variance (deterministic), handles transmittance and inscatter in a single pass.
- **Limitations:** Fixed step size may alias in high-frequency density fields. No multiple scattering. The "Frostbite" integration approximation may introduce errors if $\sigma_t'$ varies significantly within a step [G].
- **When to use:** Real-time rendering of uniform or slowly varying volumes where performance is critical and multiple scattering is negligible.

## Background
The method originates from the Frostbite engine, used in games like *Battlefield* [S1, unofficial]. It was popularized by the "Frostbite Engine" technical presentations and subsequent community implementations. The specific integration scheme is an approximation of the volumetric rendering equation optimized for real-time performance.

## Sources
[S1] al-ro/volumetrics, https://github.com/al-ro/volumetrics, fetched 2026-10-27 (unofficial)
