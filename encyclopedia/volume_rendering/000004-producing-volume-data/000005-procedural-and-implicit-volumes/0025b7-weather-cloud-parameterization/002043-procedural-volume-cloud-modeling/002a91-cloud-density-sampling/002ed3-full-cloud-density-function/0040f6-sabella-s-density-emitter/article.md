# Sabella's Density Emitter

Also called: Sabella's Method

## Overview
Sabella's Density Emitter is a volume rendering technique that models the density of particles as a continuous function rather than discrete entities, allowing for varying light intensity along a ray [S1, unofficial]. It distinguishes itself from Blinn’s fixed-intensity method by introducing a density-dependent emission term, where the brightness contribution of each sample is modulated by the local density value [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A parameterized density field $\rho(x, y, z)$, ray start $t_1$, ray end $t_2$, and attenuation parameter $\beta$ [S1, unofficial].
*   **Output:** The calculated brightness (intensity) reaching the eye [S1, unofficial].

**Data Structures:**
The method operates on a continuous density field $\rho$ [S1, unofficial]. In implementation, this is typically sampled at discrete intervals along a ray [G]. Key statistics accumulated during ray traversal include:
*   $M$: The maximum (peak) density value encountered along the ray [S1, unofficial].
*   $D$: The distance (parameter $t$) at which the peak value $M$ is encountered [S1, unofficial].
*   $C$: The center of gravity of the density distribution along the ray [S1, unofficial].
*   $R$: The total attenuation integral along the ray, defined as $R = \int_{t_1}^{t_2} \rho(\tau) d\tau$ [S1, unofficial].

**Formulas:**
The brightness $B$ is calculated by integrating the emission along the ray path:

$$ B = \int_{t_1}^{t_2} \frac{R}{\rho(t)} e^{-\int_{t_1}^{t} \rho(\tau) d\tau} dt $$

Where:
*   $\rho(t)$ is the parameterized density field $\rho(x(t), y(t), z(t))$ [S1, unofficial].
*   $R$ is the total attenuation integral $\int_{t_1}^{t_2} \rho(\tau) d\tau$ [S1, unofficial].
*   $\beta$ is a parameter value to adjust the rate of attenuation [S1, unofficial].

**Steps:**
1.  Define the ray path from $t_1$ to $t_2$ [S1, unofficial].
2.  Sample the density field $\rho(t)$ along the ray [S1, unofficial].
3.  Compute the total attenuation $R$ over the full ray segment $[t_1, t_2]$ [S1, unofficial].
4.  Compute the partial attenuation integral $\int_{t_1}^{t} \rho(\tau) d\tau$ for each point $t$ along the ray [S1, unofficial].
5.  Evaluate the integrand $\frac{R}{\rho(t)} e^{-\int_{t_1}^{t} \rho(\tau) d\tau}$ at each sample point [S1, unofficial].
6.  Integrate the result over $t$ from $t_1$ to $t_2$ to obtain Brightness [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def point_at_t(t, ray_start, ray_end):
    """Returns the 3D position at parameter t along the ray from ray_start to ray_end."""
    return ray_start + t * (ray_end - ray_start)

def sabella_density_emitter(ray_start, ray_end, density_field, num_samples):
    """
    Calculates brightness using Sabella's Density Emitter method.
    
    Args:
        ray_start: 3D start point of the ray.
        ray_end: 3D end point of the ray.
        density_field: Function rho(x,y,z) returning density at a 3D point.
        num_samples: Number of discrete samples along the ray.
        
    Returns:
        brightness: The calculated intensity.
    """
    dt = (ray_end - ray_start) / num_samples
    t_values = [ray_start + i * dt for i in range(num_samples + 1)]
    
    # Sample density field along the ray
    rho_t = [density_field(point_at_t(t, ray_start, ray_end)) for t in t_values]
    
    # Compute total attenuation integral R = integral_{t1}^{t2} rho(tau) dtau
    R = 0.0
    for i in range(num_samples):
        R += (rho_t[i] + rho_t[i+1]) / 2.0 * dt
        
    # Compute Brightness integral
    brightness = 0.0
    for i in range(num_samples):
        t_mid = (t_values[i] + t_values[i+1]) / 2.0
        rho_mid = (rho_t[i] + rho_t[i+1]) / 2.0
        
        # Compute partial attenuation integral up to t_mid
        partial_integral = 0.0
        for j in range(i + 1):
            if j < num_samples:
                partial_integral += rho_t[j] * dt
                
        # Guard against division by zero
        if rho_mid < 1e-6:
            term = 0.0
        else:
            term = (R / rho_mid) * math.exp(-partial_integral)
            
        brightness += term * dt
        
    return brightness
```

**Edge Cases and Pitfalls:**
*   **Division by Zero:** The term $\frac{R}{\rho(t)}$ requires $\rho(t) \neq 0$. If the density field contains zero values, a guard clause or epsilon offset is required [G].
*   **Precision:** The nested integrals (one for attenuation, one for brightness) can accumulate floating-point errors. High-resolution sampling is recommended [G].
*   **Boundary Conditions:** $t_1$ and $t_2$ must be correctly identified as the entry and exit points of the volume [S1, unofficial].

## Subtopics

### [Density Emitter Model](11205-density-emitter-model.md)
Simulates light from translucent objects by treating volumetric data samples as density emitters. It sets itself apart by explicitly defining the intensity accumulation formula and mapping ray-cast statistics (max density, distance, center of gravity) to color parameters.

## Operating characteristics
*   **Complexity:** The method requires two nested integrations per ray, leading to $O(N^2)$ complexity if naive integration is used, or $O(N)$ with precomputed cumulative density arrays, where $N$ is the number of samples per ray [G].
*   **Performance:** The source does not provide specific measured performance metrics, hardware, or resolution data [S1, unofficial].
*   **Strengths:** Allows varying light intensity based on density, unlike Blinn's fixed method [S1, unofficial].
*   **Limitations:** Computationally more expensive than simple alpha blending due to the complex integrand [G].

## Background
Sabella's method was presented by Sabella in 1988 [S1, unofficial]. It is a variation on the general ray-casting theme and is similar to Blinn's method [Bli82], but improves upon it by allowing varying densities where Blinn's method is fixed [S1, unofficial]. The method is discussed in the context of forward projection and volume rendering overviews [S1, unofficial].

## Sources
[S1] An Overview of Rendering from Volume Data --- including Surface and Volume Rendering, https://core.ac.uk/download/63614.pdf, fetched 2026-10-30 (unofficial)
