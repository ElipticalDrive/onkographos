# Physical Atmosphere Color Calculation

Also called: Physical Atmosphere Color, getAtmosphereColorPhysical

## Overview
Calculates the physical color of the sky and sun disk for a given view direction, accounting for Rayleigh and Mie scattering, optical depth, and phase functions [S1, unofficial]. It differs from sibling methods by providing a closed-form analytical approximation of the atmospheric radiance rather than using Monte Carlo integration or precomputed lookup tables [S1, unofficial].

## How it works
**Inputs:**
- `dir`: View direction vector (normalized).
- `sunDir`: Sun direction vector (normalized).
- `sunIntensity`: Scalar intensity of the sun.
- `turbidity`: Atmospheric turbidity parameter (default 10.0) [S1, unofficial].

**Outputs:**
- `localcolor`: The final tone-mapped atmospheric color (RGB).

**Data Structures:**
- Constants are hardcoded vectors/scalars as defined below.

**Parameters:**
- `rayleigh_total`: `vec3(5.804542996261093e-6, 1.3562911419845635e-5, 3.0265902468824876e-5)` [S1, unofficial].
- `mie_const`: `vec3(1.839991851443397, 2.779802391966052, 4.079047954386109)` [S1, unofficial].
- `mie`: `0.005` [S1, unofficial].
- `mie_directional`: `0.8` [S1, unofficial].
- `sun_angular_cos`: `0.999956676946448` [S1, unofficial].

**Steps:**

1.  **Calculate Scattering Coefficients:**
    Compute the turbidity factor $c$:
    $$ c = (0.2 \times \text{turbidity}) \times 10^{-18} $$
    Calculate Mie scattering coefficient $\beta_M$:
    $$ \beta_M = \text{vec3}(0.434 \times c \times \text{mie\_const} \times \text{mie}) $$
    Calculate sun fade factor:
    $$ \text{sunFade} = 1.0 - \text{clamp}\left(1.0 - \exp\left(\frac{\text{SunLocation.y}}{450000.0}\right), 0.0, 1.0\right) $$
    *Note: `SunLocation.y` is not defined in the provided notes. [G] Assume `SunLocation.y` refers to the y-component of the sun direction or a related altitude parameter. If undefined, this step cannot be implemented exactly.*
    Calculate Rayleigh scattering coefficient $\beta_R$:
    $$ \beta_R = \text{vec3}(\text{rayleigh\_total} \times (\text{rayleigh} - 1.0 + \text{sunFade})) $$
    where $\text{rayleigh} = 2.0$ [S1, unofficial].

2.  **Calculate Optical Path Lengths:**
    Calculate zenith angle $\theta_z$:
    $$ \theta_z = \arccos(\max(0.0, \text{dir}.y)) $$
    Calculate inverse air mass $m^{-1}$:
    $$ \text{inverse} = \frac{1.0}{\cos(\theta_z) + 0.15 \times \left(93.885 - \frac{\theta_z \times 180.0}{\pi}\right)^{-1.253}} $$
    Calculate Rayleigh optical path length $s_R$:
    $$ s_R = 8400.0 \times \text{inverse} $$
    Calculate Mie optical path length $s_M$:
    $$ s_M = 1250.0 \times \text{inverse} $$

3.  **Calculate Extinction Factor:**
    $$ f_{ex} = \exp\left(-\beta_R \times s_R + \beta_M \times s_M\right) $$
    *Note: The formula in the source uses `+ BetaM * sM` in the exponent, which is physically unusual for extinction (usually both terms subtract). We follow the source exactly.* [S1, unofficial].

4.  **Calculate Phase Functions:**
    Calculate cosine of scattering angle:
    $$ \cos\Theta = \text{dot}(\text{sunDir}, \text{dir}) $$
    Calculate Rayleigh phase term $P_R$:
    $$ P_R = \frac{3 \times (1.0 + \cos\Theta \times \cos\Theta)}{16.0 \times \pi} $$
    Calculate Mie phase term $P_M$ using Henyey-Greenstein:
    $$ P_M = \text{HenyeyGreenstein}(\text{dir}, \text{sunDir}, \text{mie\_directional}) $$
    *Note: The Henyey-Greenstein function is not defined in the notes. [G] Standard implementation: $P_M = \frac{1-g^2}{4\pi (1+g^2-2g\cos\Theta)^{3/2}}$ where $g = \text{mie\_directional}$.*

5.  **Calculate In-Scattered Light:**
    Calculate intermediate scattering terms:
    $$ \beta_R\Theta = \beta_R \times P_R $$
    $$ \beta_M\Theta = \beta_M \times P_M $$
    Calculate combined scattering ratio:
    $$ \beta_{sum} = \beta_R + \beta_M $$
    $$ \beta_{mix} = \frac{\beta_R\Theta + \beta_M\Theta}{\beta_{sum}} $$
    Calculate sun intensity factor:
    $$ \text{sunE} = \text{sunIntensity} \times \text{calcSunIntensity}() $$
    *Note: `calcSunIntensity()` is not defined. [G] Assume it returns 1.0 or a standard normalization factor.*
    Calculate primary in-scattered radiance $L_{in}$:
    $$ L_{in} = \text{pow}\left(\text{sunE} \times \beta_{mix} \times (1.0 - f_{ex}), \text{vec3}(1.5)\right) $$
    Calculate horizon fade factor $yDot$:
    $$ yDot = 1.0 - \text{sunDir}.y $$
    $$ yDot = yDot \times yDot \times yDot \times yDot \times yDot $$
    Apply horizon blending:
    $$ L_{in} = L_{in} \times \text{mix}\left(\text{vec3}(1), \text{pow}\left(\text{sunE} \times \beta_{mix} \times f_{ex}, \text{vec3}(0.5)\right), \text{clamp}(yDot, 0.0, 1.0)\right) $$

6.  **Calculate Direct Sun Disk:**
    Initialize direct light $L_0$:
    $$ L_0 = \text{vec3}(0.1 \times f_{ex}) $$
    Calculate sun disk visibility:
    $$ \text{sunDisk} = \text{smoothstep}(\text{sun\_angular\_cos}, \text{sun\_angular\_cos} + 0.00002, \cos\Theta) $$
    Add sun disk radiance:
    $$ L_0 = L_0 + (\text{sunE} \times 15000.0 \times f_{ex}) \times \text{sunDisk} $$

7.  **Combine and Tone-Map:**
    $$ \text{localcolor} = (L_{in} + L_0) \times 0.04 + \text{vec3}(0.0, 0.0003, 0.00075) $$
    Return `localcolor`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def rayleigh_phase(cos_theta):
    return 3.0 * (1.0 + cos_theta * cos_theta) / (16.0 * math.pi)

def henyey_greenstein(dir_vec, sun_dir, g):
    cos_theta = math.dot(dir_vec, sun_dir)
    # Standard Henyey-Greenstein phase function
    numerator = 1.0 - g * g
    denominator = 1.0 + g * g - 2.0 * g * cos_theta
    if denominator < 1e-8:
        denominator = 1e-8
    return numerator / (4.0 * math.pi * (denominator ** 1.5))

def get_atmosphere_color_physical(dir, sun_dir, sun_intensity, turbidity=10.0):
    # Constants
    rayleigh_total = vec3(5.804542996261093e-6, 1.3562911419845635e-5, 3.0265902468824876e-5)
    mie_const = vec3(1.839991851443397, 2.779802391966052, 4.079047954386109)
    mie = 0.005
    mie_directional = 0.8
    rayleigh = 2.0
    sun_angular_cos = 0.999956676946448
    
    # 1. Scattering Coefficients
    c = (0.2 * turbidity) * 1e-18
    beta_m = vec3(0.434 * c * mie_const * mie)
    
    # sunFade calculation requires SunLocation.y which is not defined in notes
    # [G] Placeholder for undefined variable. Assuming SunLocation.y is sun_dir.y for implementation
    sun_location_y = sun_dir.y 
    sun_fade = 1.0 - clamp(1.0 - math.exp(sun_location_y / 450000.0), 0.0, 1.0)
    
    beta_r = rayleigh_total * (rayleigh - 1.0 + sun_fade)
    
    # 2. Optical Path Lengths
    zenith = math.acos(max(0.0, dir.y))
    zenith_deg = zenith * 180.0 / math.pi
    inverse = 1.0 / (math.cos(zenith) + 0.15 * (93.885 - zenith_deg) ** -1.253)
    s_r = 8400.0 * inverse
    s_m = 1250.0 * inverse
    
    # 3. Extinction Factor
    fex = math.exp(-beta_r * s_r + beta_m * s_m)
    
    # 4. Phase Functions
    cos_theta = math.dot(sun_dir, dir)
    r_phase = rayleigh_phase(cos_theta)
    m_phase = henyey_greenstein(dir, sun_dir, mie_directional)
    
    beta_r_theta = beta_r * r_phase
    beta_m_theta = beta_m * m_phase
    
    beta_sum = beta_r + beta_m
    beta_mix = (beta_r_theta + beta_m_theta) / beta_sum
    
    # sunIntensity calculation placeholder
    # [G] Assuming calcSunIntensity() returns 1.0
    sun_e = sun_intensity * 1.0
    
    # 5. In-Scattered Light
    lin = (sun_e * beta_mix * (1.0 - fex)) ** 1.5
    
    y_dot = 1.0 - sun_dir.y
    y_dot = y_dot ** 5
    
    lin = lin * mix(vec3(1.0), (sun_e * beta_mix * fex) ** 0.5, clamp(y_dot, 0.0, 1.0))
    
    # 6. Direct Sun Disk
    l0 = vec3(0.1 * fex)
    sun_disk = smoothstep(sun_angular_cos, sun_angular_cos + 0.00002, cos_theta)
    l0 = l0 + (sun_e * 15000.0 * fex) * sun_disk
    
    # 7. Combine and Tone-Map
    localcolor = (lin + l0) * 0.04 + vec3(0.0, 0.0003, 0.00075)
    
    return localcolor

def vec3(x, y, z):
    return (x, y, z)

def clamp(val, min_val, max_val):
    return max(min_val, min(max_val, val))

def mix(a, b, t):
    return a + t * (b - a)

def smoothstep(edge0, edge1, x):
    t = clamp((x - edge0) / (edge1 - edge0), 0.0, 1.0)
    return t * t * (3.0 - 2.0 * t)

def math_dot(v1, v2):
    return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]

def math_exp(x):
    return math.exp(x)

def math_acos(x):
    return math.acos(x)

def math_cos(x):
    return math.cos(x)

def math_pow(x, y):
    return x ** y
```

## Subtopics

### Ray Marching with In/Out-Scattering and Density Scaling
A method that uses ray marching to compute in-scattering and out-scattering with density scaling. Choose this when volumetric density varies significantly within the scene, unlike the homogeneous atmosphere assumption here. [2318-ray-marching-with-in-out-scattering-and-density-scaling.md]

### Single Scattering Approximation with Precomputed Terms
Uses precomputed terms to approximate single scattering. Choose this for performance when the sun position or view direction changes frequently, avoiding per-pixel phase function calculations. [4188-single-scattering-approximation-with-precomputed-terms.md]

### Analytical Approximation of Single Scattering Radiance
Provides an analytical approximation for single scattering radiance. Choose this when a simpler, faster approximation is acceptable compared to the more detailed physical model here. [4244-analytical-approximation-of-single-scattering-radiance.md]

### Volume Single Scattering (Monte Carlo with Sphere Light)
Uses Monte Carlo integration with a sphere light. Choose this when the light source has a significant angular size, unlike the point light assumption in the parent method. [4549-volume-single-scattering-monte-carlo-with-sphere-light.md]

### Volume Single Scattering (Uniform Ray Marching with Point Light Background)
Uses uniform ray marching with a point light background. Choose this for simpler implementations where Monte Carlo variance is undesirable. [4550-volume-single-scattering-uniform-ray-marching-with-point-lig.md]

### BSSRDF Approximation with Dipole and Single Scattering
Combines BSSRDF dipole approximation with single scattering. Choose this for subsurface scattering effects in participating media, which this method does not handle. [4750-bssrdf-approximation-with-dipole-and-single-scattering.md]

## Operating characteristics
- **Time Complexity:** O(1) per pixel, as it is a closed-form analytical calculation.
- **Memory Complexity:** O(1), requiring only constant storage for coefficients.
- **Strengths:** Extremely fast, no sampling noise, physically based parameters (turbidity, scattering coefficients).
- **Limitations:** Assumes a homogeneous atmosphere; does not account for multiple scattering; relies on undefined variables (`SunLocation.y`, `calcSunIntensity()`) which may require implementation-specific adjustments.
- **When to use:** Real-time sky rendering where performance is critical and the atmosphere can be approximated as homogeneous.
- **When not to use:** Scenes with complex, non-homogeneous volumetric density or when multiple scattering effects are significant.

## Background
Originated from the `erickTornero/realtime-volumetric-cloudscapes` repository [S1, unofficial]. It is used in real-time graphics for sky and atmosphere rendering.

## Sources
[S1] erickTornero/realtime-volumetric-cloudscapes, https://github.com/erickTornero/realtime-volumetric-cloudscapes, fetched 2026-10-27 (unofficial)
