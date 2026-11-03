# Global Illumination via Hemisphere Sampling

Also called: Diffuse GI, Sky GI

## Overview
A global illumination solution that models single-bounce light contribution from the sky by creating a virtual skybox of infinite directional lights and sampling it over the hemisphere directed away from the surface normal [S1]. Unlike typical raster renderers that model the sky as one or more directional lights (causing inaccurate shadowing and dark scenes), this method provides accurate shadowing and illumination by integrating radiance over the hemisphere [S1].

## How it works
**Inputs:** Surface position $p$, surface normal $n$, surface roughness $\alpha$ (for specular variant), temporal samples $temporal\_x, temporal\_y \in [0, 1]$, sky radiance function $s_{sky}(\omega_i)$, shadow hardness parameter $k$ [S1].
**Outputs:** Diffuse Global Illumination color contribution $GI_{diffuse}$ and/or Specular Global Illumination color contribution $GI_{specular}$ [S1].

**Data Structures:**
*   `temporal_x`: Float, temporal sample X coordinate [S1].
*   `temporal_y`: Float, temporal sample Y coordinate [S1].
*   `n`: Normalized 3D vector, surface normal [S1].
*   `T`: 3D vector, tangent basis vector [S1].
*   `B`: 3D vector, bitangent basis vector [S1].
*   `H`: 3D vector, hemisphere sample direction in local space [S1].
*   `omega_i`: 3D vector, world-space sample direction [S1].

**Parameters:**
*   Shadow hardness $k=3$ for diffuse GI [S1].
*   Shadow hardness $k = 3 + 1/\alpha$ for specular GI [S1].

**Steps:**

1.  **Construct Tangent Basis:**
    Calculate the tangent vector $T$ to avoid degeneracy when the normal is aligned with the Z-axis [S1]:
    $$
    T = \begin{cases} 
    \{0, 0, 1\} \times n & \text{if } |n_z| < 0.999 \\
    \{1, 0, 0\} \times n & \text{otherwise}
    \end{cases}
    $$
    Calculate the bitangent $B$:
    $$
    B = n \times T
    $$
    [S1]

2.  **Sample Direction:**
    Depending on the variant (Diffuse or Specular), compute the sample direction $\omega_i$ using the respective sampling function [S1].

    *For Diffuse GI (Uniform Hemisphere Sampling):*
    Compute local hemisphere coordinates $H$:
    $$
    \phi = 2\pi \cdot temporal\_x
    $$
    $$
    \theta = 0.5\pi \cdot temporal\_y
    $$
    $$
    H = \{\cos\phi \sin\theta, \sin\phi \sin\theta, \cos\theta\}
    $$
    Transform to world space:
    $$
    \omega_i = \{T, B, n\} \cdot H
    $$
    [S1]

    *For Specular GI (GGX Importance Sampling):*
    Compute $\cos\theta$ based on roughness $\alpha$:
    $$
    \cos\theta = \frac{1 - temporal\_x}{1 + (\alpha^2 - 1) temporal\_y}
    $$
    Derive $\sin\theta = \sqrt{1 - \cos^2\theta}$ [S1].
    Compute $\phi = 2\pi \cdot temporal\_x$.
    Compute local hemisphere coordinates $H$:
    $$
    H = \{\cos\phi \sin\theta, \sin\phi \sin\theta, \cos\theta\}
    $$
    Transform to world space:
    $$
    \omega_i = \{T, B, n\} \cdot H
    $$
    [S1]

3.  **Calculate Soft Shadow:**
    Compute the soft shadow term $SoftMarch(p, \omega_i, k)$ [S1].

4.  **Compute Radiance Contribution:**

    *Diffuse GI:*
    $$
    GI_{diffuse} = \frac{k_d c}{\pi} \int_{\Omega} s_{sky}(\omega_i) SoftMarch(p, \omega_i, 3) (n \cdot \omega_i) d\omega_i
    $$
    In practice, this is approximated by the single sample:
    $$
    GI_{diffuse} \approx \frac{k_d c}{\pi} s_{sky}(\omega_i) SoftMarch(p, \omega_i, 3) (n \cdot \omega_i)
    $$
    where $k_d$ is the diffuse coefficient and $c$ is a scaling constant [S1].

    *Specular GI:*
    $$
    GI_{specular} = k_s \int_{\Omega} \frac{D F G}{4 (\omega_o \cdot n) (\omega_i \cdot n)} s_{sky}(\omega_i) SoftMarch(p, \omega_i, 3 + \frac{1}{\alpha}) (n \cdot \omega_i) d\omega_i
    $$
    In practice, approximated by the single sample:
    $$
    GI_{specular} \approx k_s \frac{D F G}{4 (\omega_o \cdot n) (\omega_i \cdot n)} s_{sky}(\omega_i) SoftMarch(p, \omega_i, 3 + \frac{1}{\alpha}) (n \cdot \omega_i)
    $$
    where $k_s$ is the specular coefficient, $D, F, G$ are Cook-Torrance BRDF terms, and $\omega_o$ is the view direction [S1].

5.  **Accumulate:**
    Add $GI_{diffuse}$ to the pixel color from the first shading pass [S1].
    Add $GI_{specular}$ to the pixel color computed from the first shading pass and diffuse GI pass [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_hemisphere_gi(p, n, alpha, temporal_x, temporal_y, s_sky, k, kd, c, ks, D, F, G, wo):
    # 1. Construct Tangent Basis
    if abs(n[2]) < 0.999:
        T = cross([0, 0, 1], n)
    else:
        T = cross([1, 0, 0], n)
    B = cross(n, T)
    
    # 2. Sample Direction (Diffuse variant shown; swap logic for Specular)
    phi = 2 * math.pi * temporal_x
    theta = 0.5 * math.pi * temporal_y
    cos_theta = math.cos(theta)
    sin_theta = math.sin(theta)
    H = [math.cos(phi) * sin_theta, math.sin(phi) * sin_theta, cos_theta]
    omega_i = [dot(T, H), dot(B, H), dot(n, H)]
    
    # 3. Calculate Soft Shadow
    # SoftMarch(p, w, k): Shadow ray march. Not given in the sources.
    shadow = SoftMarch(p, omega_i, k)
    
    # 4. Compute Radiance Contribution
    ndotw = dot(n, omega_i)
    diffuse_term = (kd * c / math.pi) * s_sky(omega_i) * shadow * ndotw
    
    # Specular variant uses different k and Cook-Torrance terms
    specular_term = ks * (D * F * G) / (4 * dot(wo, n) * dot(omega_i, n)) * s_sky(omega_i) * shadow * ndotw
    
    return diffuse_term, specular_term
```

**Edge Cases and Pitfalls:**
*   If $|n_z| \ge 0.999$, the standard cross product for $T$ might be degenerate or unstable; the code switches to using the X-axis vector $\{1, 0, 0\}$ for the cross product [S1].
*   Using only one sample per frame produces a checkerboard pattern without Temporal Anti-Aliasing (TAA) [S1].
*   Relies on TAA for accurate results because it uses Monte Carlo integration over time with limited samples [S1].

## Subtopics

### Specular Global Illumination via GGX Importance Sampling
A reflection technique that samples the skybox using a specular lobe distribution based on the Cook-Torrance specular term and surface roughness $\alpha$, instead of uniform hemisphere sampling [S1]. It differs from diffuse GI by using a specular lobe distribution (GGX) dependent on roughness $\alpha$ rather than uniform hemisphere sampling [S1]. It relies on Temporal Anti-Aliasing (TAA) for accurate results because it uses Monte Carlo integration over time with limited samples [S1].

## Operating characteristics
*   **Complexity:** Computationally expensive to integrate fully over the hemisphere, so it relies on single-sample Monte Carlo integration per frame [S1].
*   **Performance:** Requires Temporal Anti-Aliasing (TAA) to converge to an accurate image within a few frames; without TAA, the result is noisy (checkerboard pattern) [S1].
*   **Strengths:** Provides accurate shadowing and illumination from the sky, avoiding the dark scenes and inaccurate shadows caused by modeling the sky as simple directional lights in typical raster renderers [S1].
*   **Limitations:** Single-sample approach introduces noise without temporal accumulation; dependent on TAA stability [S1].

## Background
This method was implemented in a real-time renderer using CUDA, aiming to achieve visual fidelity comparable to offline high-fidelity techniques like ray tracing, while surpassing typical real-time raster renderers in quality [S1]. It addresses the limitation of engines like the Source Engine, which model the sky as directional lights (sun and sky lights) that fail to contribute accurate shadowing [S1]. The approach was evaluated at 1280 × 720 on an RTX 2080 Super, comparing against Blender Cycles and UE4 [S1].

## Sources
[S1] Raymarching Distance Fields with CUDA, https://www.mdpi.com/2079-9292/10/22/2730, fetched 2026-10-27
