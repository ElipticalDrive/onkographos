# Sphere Light Shading with Importance Solid Angle Sampling

Also called: Importance Solid Angle Sampling for Sphere Lights

## Overview
This method estimates direct illumination from a spherical light source using Monte Carlo integration over the solid angle subtended by the light [S1, unofficial]. It samples directions uniformly within the cone defined by the light's extent at the shading point [S1, unofficial]. It is a specific strategy within the broader [Sphere Light Sampling with Multiple Strategies](2207-sphere-light-sampling-with-multiple-strategies.md) framework [G].

## How it works
**Inputs:**
- Shading point position $X$ [S1, unofficial].
- Light center $C$ and radius $R_{light}$ [S1, unofficial].
- Surface normal $N$ [S1, unofficial].
- Bidirectional Reflectance Distribution Function (BRDF) $f_r$ [S1, unofficial].
- Incident radiance $L_i$ from the light [S1, unofficial].
- Shadow status $inShadow$ [S1, unofficial].

**Outputs:**
- Estimated reflected radiance contribution $L_r$ [S1, unofficial].

**Data Structures:**
- Standard 3D vector types for positions, directions, and normals [G].
- Random number generator for uniform samples $\eta_1, \eta_2 \in [0, 1)$ [G].

**Steps:**
1. **Compute Geometry:** Calculate the vector from the shading point to the light center: $\vec{d} = C - X$. Compute the distance $D = |\vec{d}|$ [S1, unofficial].
2. **Determine Cone Parameters:** Calculate the sine and cosine of the maximum half-angle $\theta_{max}$ subtended by the light sphere:
   $$ \sin\theta_{max} = \frac{R_{light}}{D} $$
   $$ \cos\theta_{max} = \sqrt{1 - \sin^2\theta_{max}} $$
   [S1, unofficial].
3. **Sample Direction:** Sample a direction $\omega_i$ uniformly within the solid angle of the cone [S1, unofficial].
4. **Shadow Test:** Trace a ray from $X$ in direction $\omega_i$ with length $D$ [S1, unofficial]. If the ray hits the light sphere, set $inShadow = 0$; otherwise $inShadow = 1$ [S1, unofficial].
5. **Calculate PDF Weight:** The solid angle subtended by the light is $\Omega = 2\pi(1 - \cos\theta_{max})$ [S1, unofficial]. The PDF for uniform sampling over this solid angle is $p(\omega_i) = 1/\Omega$, so the importance sampling weight is $2\pi(1 - \cos\theta_{max})$ [S1, unofficial].
6. **Update Estimator:** Accumulate the radiance contribution:
   $$ L_r \leftarrow L_r + f_r \cdot L_i \cdot \cos\theta_i \cdot (1 - inShadow) \cdot 2\pi(1 - \cos\theta_{max}) $$
   where $\theta_i$ is the angle between $\omega_i$ and the surface normal $N$ [S1, unofficial].
7. **Average:** Repeat for $N$ samples and divide by $N$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def randf():
    return random.uniform(0.0, 1.0)

def length(v):
    return math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)

def dot(a, b):
    return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]

def cross(a, b):
    return (a[1]*b[2] - a[2]*b[1], a[2]*b[0] - a[0]*b[2], a[0]*b[1] - a[1]*b[0])

def normalize(v):
    l = length(v)
    return (v[0]/l, v[1]/l, v[2]/l)

def shade_sphere_light_importance_solid_angle(X, C, R_light, N, fr, L_i, num_samples):
    """
    Estimates direct lighting from a sphere using importance solid angle sampling.
    """
    total_radiance = 0.0
    
    # Vector from shading point to light center
    d = (C[0]-X[0], C[1]-X[1], C[2]-X[2])
    dist = length(d)
    
    # Handle degenerate case where point is inside the light
    if dist <= R_light:
        cos_theta_max = 1.0
    else:
        sin_theta_max = R_light / dist
        cos_theta_max = math.sqrt(1.0 - sin_theta_max**2)
        
    # PDF constant (inverse of solid angle)
    pdf_constant = 2.0 * math.pi * (1.0 - cos_theta_max)
    
    for _ in range(num_samples):
        # 1. Sample direction uniformly within the cone subtended by the light
        phi = 2.0 * math.pi * randf()
        cos_theta = randf() * (1.0 - cos_theta_max) + cos_theta_max
        sin_theta = math.sqrt(1.0 - cos_theta**2)
        
        # Construct local basis aligned with d
        if abs(d[0]) < 0.9:
            up = (1.0, 0.0, 0.0)
        else:
            up = (0.0, 1.0, 0.0)
        tangent = normalize(cross(d, up))
        bitangent = cross(d, tangent)
        
        omega_i = (
            sin_theta * (math.cos(phi) * tangent[0] + math.sin(phi) * bitangent[0]) + cos_theta * d[0],
            sin_theta * (math.cos(phi) * tangent[1] + math.sin(phi) * bitangent[1]) + cos_theta * d[1],
            sin_theta * (math.cos(phi) * tangent[2] + math.sin(phi) * bitangent[2]) + cos_theta * d[2]
        )
        omega_i = normalize(omega_i)
        
        # 2. Shadow test: Trace a ray from X in direction omega_i with length dist
        # Source does not provide ray tracing implementation
        in_shadow = 1.0 
        
        # 3. Calculate cosine term
        cos_theta_i = max(0.0, dot(omega_i, N))
        
        # 4. Update estimator
        radiance_contribution = fr * L_i * cos_theta_i * (1.0 - in_shadow) * pdf_constant
        total_radiance += radiance_contribution
        
    return total_radiance / num_samples
```

**Edge Cases and Pitfalls:**
- **Inside Light:** If the shading point is inside the light sphere ($D < R_{light}$), $\sin\theta_{max} > 1$, leading to NaNs in $\cos\theta_{max}$ [G].
- **Precision:** For very distant lights, $\theta_{max}$ is small, and $1 - \cos\theta_{max}$ may suffer from catastrophic cancellation [G].

## Subtopics

### Sphere Light Shading with Cosine-Weighted Sampling
Estimates direct illumination from a spherical light by sampling directions weighted by the cosine of the angle with the surface normal [S1, unofficial]. The probability density function is $p(\omega_i) = \frac{\cos\theta}{\pi}$ [S1, unofficial]. The estimator update is $L_r \leftarrow L_r + f_r \cdot L_i \cdot (1 - inShadow) \cdot \pi$ [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ per shading point, where $N$ is the number of samples [G]. Each sample requires a shadow test [G].
- **Memory Complexity:** $O(1)$ auxiliary memory [G].
- **Strengths:** Unbiased estimator; handles arbitrary light positions and sizes [G].
- **Limitations:** Requires a shadow test for each sample [G].

## Background
This method is derived from standard Monte Carlo integration techniques for light transport [S1, unofficial]. It is implemented in the `VolumeSingleScattering` repository by wangyuehi [S1, unofficial].

## Sources
[S1] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
