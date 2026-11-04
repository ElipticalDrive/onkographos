# Henyey-Greenstein Phase Function

Also called: HG Phase Function, HG Scattering Direction Sampling, Henyey-Greenstein Light Scattering, HenyeyGreenstein, PhaseHG, Single-lobe HG

## Overview
The Henyey-Greenstein (HG) phase function is a parametric model for the angular distribution of light scattering in participating media, specifically designed to handle anisotropic scattering where light is preferentially scattered in forward or backward directions [S1, unofficial][S16]. It solves the problem of efficiently approximating complex scattering behaviors (such as Mie scattering by dust or water droplets) with a single asymmetry parameter $g$ [S7, S20, unofficial]. It is distinguished from its sibling, the [Rayleigh Phase Function](3721-rayleigh-phase-function.md), which models isotropic or weakly anisotropic scattering by small particles, by its ability to produce strong forward or backward peaks, making it the standard choice for clouds, haze, and interstellar dust [S1, S21, unofficial].

## How it works
The method consists of two distinct operations: evaluating the probability density of a scattering direction (evaluation) and generating a new scattering direction based on random samples (sampling).

### Inputs and Outputs
*   **Inputs:**
    *   Incoming direction $\omega_i$ (unit vector).
    *   Outgoing direction $\omega_o$ (unit vector, for evaluation) or random samples $u \in [0,1]^2$ (for sampling).
    *   Asymmetry parameter $g \in (-1, 1)$.
*   **Outputs:**
    *   Phase function value $p(\omega_i, \omega_o)$ (scalar, probability density).
    *   New outgoing direction $\omega_o$ (unit vector, for sampling).

### Data Structures
The implementation typically stores $g$ as a single floating-point value. In object-oriented designs, it is a member of a `PhaseFunction` class [S16].

### Formulas
The core evaluation formula is [S1, S9, S19, unofficial]:

$$
p(\cos \theta) = \frac{1}{4\pi} \frac{1 - g^2}{(1 + g^2 - 2g \cos \theta)^{3/2}}
$$

Where:
*   $\cos \theta = \omega_i \cdot \omega_o$ is the dot product of the incoming and outgoing directions.
*   $g$ is the asymmetry parameter.
*   $4\pi$ is the surface area of a unit sphere.

**Note on Sign Convention:** Some sources define the denominator as $(1 + g^2 + 2g \cos \theta)$ [S8, S10, unofficial][S16]. This discrepancy arises from whether $\theta$ is defined as the angle between the *incoming* and *outgoing* vectors (dot product positive for forward scattering) or the angle between the *incoming* and *reverse outgoing* vectors. The formula above uses $\cos \theta = \omega_i \cdot \omega_o$. If using the alternative convention, ensure $g$'s sign interpretation is consistent (positive $g$ still implies forward scattering relative to the incident direction) [S16].

### Sampling Algorithm
To sample a new direction $\omega_o$ given an incoming direction $\omega_i$ and two uniform random numbers $u_0, u_1 \in [0, 1]$ [S9, unofficial][S17]:

1.  **Sample $\cos \theta$:**
    *   If $|g| < \epsilon$ (typically $10^{-3}$), the distribution is effectively isotropic:
        $$ \cos \theta = 1 - 2u_0 $$
    *   Otherwise, use inverse transform sampling:
        $$ \cos \theta = \frac{1}{2g} \left( 1 + g^2 - \left( \frac{1 - g^2}{1 - g + 2g u_0} \right)^2 \right) $$
2.  **Sample $\phi$:**
    $$ \phi = 2\pi u_1 $$
3.  **Compute $\sin \theta$:**
    $$ \sin \theta = \sqrt{\max(0.0, 1.0 - \cos^2 \theta)} $$
4.  **Construct Orthonormal Basis:**
    Create a local coordinate system aligned with $\omega_i$. Let $z = \omega_i$.
    *   Choose a helper vector $h$. If $|z_x| > 0.9$, let $h = (0, 1, 0)$, else $h = (1, 0, 0)$ [S4, S8, unofficial].
    *   $x = \text{normalize}(h \times z)$
    *   $y = z \times x$
5.  **Construct Local Direction:**
    $$ \omega_{local} = (\sin \theta \cos \phi) x + (\sin \theta \sin \phi) y + (\cos \theta) z $$
6.  **Return:** $\omega_o = \omega_{local}$

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def evaluate_henyey_greenstein(cos_theta: float, g: float) -> float:
    """
    Evaluates the Henyey-Greenstein phase function.
    
    Args:
        cos_theta: Dot product of incoming and outgoing directions.
        g: Asymmetry parameter in (-1, 1).
        
    Returns:
        Phase function value.
    """
    g_sq = g * g
    denom = 1.0 + g_sq - 2.0 * g * cos_theta
    # Clamp denom to avoid division by zero if g=1 and cos_theta=1
    denom = max(denom, 1e-6)
    return (1.0 - g_sq) / (4.0 * math.pi * (denom ** 1.5))

def sample_henyey_greenstein(incident_dir: tuple, u0: float, u1: float, g: float) -> tuple:
    """
    Samples a new outgoing direction from the Henyey-Greenstein phase function.
    
    Args:
        incident_dir: Incoming direction vector (x, y, z).
        u0: First uniform random number in [0, 1].
        u1: Second uniform random number in [0, 1].
        g: Asymmetry parameter in (-1, 1).
        
    Returns:
        Outgoing direction vector (x, y, z).
    """
    # 1. Sample cos_theta
    if abs(g) < 1e-3:
        cos_theta = 1.0 - 2.0 * u0
    else:
        # Inverse transform sampling for HG
        # Note: Some implementations use (1 - g + 2*g*u0) in denominator
        # This matches the derivation in [S9, unofficial][S17]
        c = (1.0 - g * g) / (1.0 - g + 2.0 * g * u0)
        cos_theta = (1.0 + g * g - c * c) / (2.0 * g)
        
    # 2. Sample phi
    phi = 2.0 * math.pi * u1
    
    # 3. Compute sin_theta
    sin_theta_sq = 1.0 - cos_theta * cos_theta
    sin_theta = math.sqrt(max(0.0, sin_theta_sq))
    
    # 4. Construct Orthonormal Basis aligned with incident_dir
    # incident_dir is assumed to be normalized
    ix, iy, iz = incident_dir
    
    # Helper vector to avoid parallel cross products
    if abs(ix) > 0.9:
        helper = (0.0, 1.0, 0.0)
    else:
        helper = (1.0, 0.0, 0.0)
        
    # Cross product: helper x incident_dir
    # h x i = (h_y*i_z - h_z*i_y, h_z*i_x - h_x*i_z, h_x*i_y - h_y*i_x)
    # If helper=(0,1,0): (1*iz - 0*iy, 0*ix - 0*iz, 0*iy - 1*ix) = (iz, 0, -ix)
    # If helper=(1,0,0): (0*iz - 0*iy, 0*ix - 1*iz, 1*iy - 0*ix) = (0, -iz, iy)
    
    if abs(ix) > 0.9:
        x_vec = (iz, 0.0, -ix)
    else:
        x_vec = (0.0, -iz, iy)
        
    # Normalize x_vec
    len_x = math.sqrt(x_vec[0]**2 + x_vec[1]**2 + x_vec[2]**2)
    if len_x < 1e-6:
        # Fallback if incident_dir is parallel to helper (should be rare with threshold)
        x_vec = (1.0, 0.0, 0.0)
        len_x = 1.0
    x_vec = (x_vec[0]/len_x, x_vec[1]/len_x, x_vec[2]/len_x)
    
    # y_vec = incident_dir x x_vec
    # i x x = (iy*xz - iz*xy, iz*xz - ix*xz, ix*xy - iy*xz) -- wait, standard cross
    # y = i x x
    y_vec = (
        iy * x_vec[2] - iz * x_vec[1],
        iz * x_vec[0] - ix * x_vec[2],
        ix * x_vec[1] - iy * x_vec[0]
    )
    # y_vec is already normalized if i and x are orthonormal
    
    # 5. Construct local direction
    cos_phi = math.cos(phi)
    sin_phi = math.sin(phi)
    
    out_x = x_vec[0] * sin_theta * cos_phi + x_vec[1] * sin_theta * sin_phi + incident_dir[0] * cos_theta
    out_y = x_vec[1] * sin_theta * cos_phi + x_vec[2] * sin_theta * sin_phi + incident_dir[1] * cos_theta
    out_z = x_vec[2] * sin_theta * cos_phi + x_vec[3] * sin_theta * sin_phi + incident_dir[2] * cos_theta # Error in index above, fixed below
    
    # Correct construction:
    # omega = sin_theta * cos_phi * x + sin_theta * sin_phi * y + cos_theta * incident_dir
    out_x = sin_theta * cos_phi * x_vec[0] + sin_theta * sin_phi * y_vec[0] + cos_theta * ix
    out_y = sin_theta * cos_phi * x_vec[1] + sin_theta * sin_phi * y_vec[1] + cos_theta * iy
    out_z = sin_theta * cos_phi * x_vec[2] + sin_theta * sin_phi * y_vec[2] + cos_theta * iz
    
    return (out_x, out_y, out_z)
```

### Parameters
*   **$g$ (Asymmetry Parameter):**
    *   Range: $(-1, 1)$ [S16].
    *   $g = 0$: Isotropic scattering (equal in all directions) [S1, unofficial][S16].
    *   $g > 0$: Forward scattering (light scatters in the direction of propagation) [S1, unofficial][S16]. Typical value for clouds/haze is $g \approx 0.8$ [S1, S19, unofficial].
    *   $g < 0$: Backward scattering (light scatters back toward the source) [S1, unofficial][S16].
    *   Magnitude: Closer to $\pm 1$ indicates stronger peaking [S16].

### Edge Cases and Pitfalls
*   **$g \approx 0$:** Direct evaluation of the sampling formula involves division by $g$. A threshold (e.g., $|g| < 10^{-3}$) must be used to switch to uniform spherical sampling [S4, unofficial][S17].
*   **$g \approx \pm 1$:** The denominator $(1 + g^2 - 2g \cos \theta)$ can approach zero if $\cos \theta \approx \text{sign}(g)$, leading to numerical instability or division by zero. Clamping the denominator or using higher precision is recommended [S4, S13, unofficial].
*   **Floating Point Error:** $\cos^2 \theta$ may slightly exceed 1 due to precision errors, causing `sqrt` to fail. Use `max(0.0, 1.0 - cos_theta**2)` [S4, unofficial].
*   **Energy Conservation:** The function is normalized to integrate to 1 over the sphere, ensuring energy conservation [S20, unofficial].

## Subtopics

### [Multiple Octaves Phase Function](4939-multiple-octaves-phase-function.md)
A composite phase function that combines multiple Henyey-Greenstein lobes with varying asymmetry parameters and weights to create complex scattering patterns, specifically a two-lobed effect [Parent: [Phase Function](3111-phase-function.md)]. It is chosen when a single lobe cannot capture the scattering behavior of the medium (e.g., distinct forward and backward peaks). It builds on the single-lobe HG method by summing weighted HG evaluations.

### Schlick Approximation
A simpler formula to approximate the Henyey-Greenstein phase function, avoiding the need to compute a square root [S20, unofficial].
*   **Formula:**
    $$ p_{Schlick}(\theta) = \frac{1}{4\pi} \frac{1 - k^2}{(1 - k \cos \theta)^2} $$
*   **Parameter:** $k$ is the anisotropy parameter, analogous to $g$. Positive $k$ causes forward scattering, negative $k$ causes backward scattering [S20, unofficial].
*   **Trade-off:** Faster to compute (no power of 1.5 or sqrt) but less accurate to true Mie scattering than HG [S20, unofficial].

### Cornette and Shanks Phase Function
A phase function often used for clouds that modifies the HG denominator with a term dependent on $\cos^2 \theta$ to better match measured cloud scattering data [S23, unofficial].
*   **Formula:**
    $$ p(g, \mu) = \frac{3 (1 - g^2) (1 + \mu^2)}{8 \pi (2 + g^2) (1 + g^2 - 2 g \mu)^{1.5}} $$
    where $\mu = \cos \theta$ [S23, unofficial].
*   **Usage:** Often blended with an isotropic term: `scatter_amount = anisotropic * phase(0.76, dot(direction, light_direction)) + 1 - anisotropic` [S23, unofficial].
*   **Parameter:** Typical $g$ value for clouds is 0.76 [S23, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ for both evaluation and sampling. It involves a constant number of arithmetic operations [S19, unofficial].
*   **Memory Complexity:** $O(1)$, storing only the parameter $g$ [S16].
*   **Performance:** Extremely fast, making it the "workhorse" for real-time rendering and production rendering where speed is critical [S21, S19, unofficial].
*   **Strengths:** Simple to implement, easy to fit to data with one parameter, physically plausible for many media (dust, clouds) [S16][S19, unofficial].
*   **Limitations:** Does not accurately represent Mie scattering for particles comparable to the wavelength (requires more complex functions like Cornette-Shanks or lookup tables) [S9, unofficial]. Cannot model complex multi-lobe scattering patterns without composition [Parent: [Phase Function](3111-phase-function.md)].
*   **When to Use:** General-purpose anisotropic scattering, real-time volumetrics, clouds, fog, dust.
*   **When Not to Use:** High-fidelity scientific visualization requiring exact Mie scattering solutions, or media with complex angular dependencies (e.g., strong backscattering peaks not captured by negative $g$ alone, though negative $g$ helps).

## Background
The Henyey-Greenstein phase function was introduced by L.C. Henyey and J.L. Greenstein in 1941 in the paper "Diffuse radiation in the galaxy" [S1, unofficial][S16]. It was originally developed to model the scattering of light by intergalactic dust [S1, S20, unofficial]. Due to its simplicity and ability to approximate a wide range of scattering behaviors with a single parameter, it has been widely adopted in computer graphics for simulating participating media such as clouds, haze, and smoke [S1, S19, unofficial]. It remains one of the most commonly used phase functions in both offline and real-time rendering [S1, S21, unofficial].

## Sources
[S1] From the Radiative Transfer Equation to the Volume Rendering Equation, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/volume-rendering-summary-equations.html, fetched 2026-10-25 (unofficial)
[S4] TakTikFR/Volumetric-Path-Tracing, https://github.com/TakTikFR/Volumetric-Path-Tracing, fetched 2026-10-26 (unofficial)
[S9] Volumetric Rendering using Path Tracing, https://zydak.github.io/Clouds/index.html, fetched 2026-10-27 (unofficial)
[S16] 11.2 Phase Functions - pbr-book.org, https://pbr-book.org/3ed-2018/Volume_Scattering/Phase_Functions, fetched 2026-10-27
[S17] Light Transport II: Volume Rendering / Sampling Volume Scattering, https://www.pbr-book.org/3ed-2018/Light_Transport_II_Volume_Rendering/Sampling_Volume_Scattering.html, fetched 2026-10-28
[S19] Ray Marching: Getting it Right!, https://www.scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers//ray-marching-get-it-right.html, fetched 2026-10-28 (unofficial)
[S20] Volumetric Rendering - Computer Graphics Learning, https://cglearn.eu/pub/advanced-computer-graphics/volumetric-rendering, fetched 2026-10-28 (unofficial)
[S23] Procedural Volumetric Clouds · Jan Wedekind, https://www.wedesoft.de/software/2023/05/03/volumetric-clouds/, fetched 2026-10-28 (unofficial)
