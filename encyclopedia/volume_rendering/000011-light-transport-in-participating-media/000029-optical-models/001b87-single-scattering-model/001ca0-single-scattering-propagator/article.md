# Single Scattering Propagator

Also called: Direct Light Propagator G_d

## Overview
The Single Scattering Propagator, denoted as $G_{single}$ or $G_d$, models light transport that has undergone zero or exactly one scattering event within a participating medium [S1]. It serves as the "direct" component of the total light transport propagator, separating unscattered and singly-scattered light from multiply-scattered (indirect) light [S1]. It is distinguished from the Utah Approximation by its rigorous inclusion of volume attenuation and phase function integration, whereas the Utah Approximation ignores occlusion and attenuation between the light source and the voxel [1717-utah-approximation-single-scattering.md].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Start position $\mathbf{x}$, start direction $\vec{\omega}$, end position $\mathbf{x}'$, end direction $\vec{\omega}'$, total path length $s$, scattering coefficient $b(\mathbf{x})$, phase function $P(\mathbf{x}, \vec{\omega}_{in}, \vec{\omega}_{out})$, and the no-scatter propagator $G_{noscatter}$ [S1].
*   **Outputs:** The propagator value $G_{single}(s, \mathbf{x}, \vec{\omega}, \mathbf{x}', \vec{\omega}')$ [S1].

**Data Structures:**
The propagator is a mathematical operator mapping spatial and angular coordinates to radiance contribution [S1]. Implementation typically uses floating-point values for $b$ and $P$ [G].

**Method Steps:**
The calculation of $G_{single}$ involves integrating over all possible intermediate points $\mathbf{x}''$ where a single scattering event occurs [S1]. The process corresponds to the standard single scattering ray marching algorithm [S1]:

1.  **Direct Transmission ($G_{noscatter}$):** Calculate the light that travels from $\mathbf{x}$ to $\mathbf{x}'$ without any scattering [S1]. This is handled by the no-scatter propagator $G_{noscatter}(s, \mathbf{x}, \vec{\omega}, \mathbf{x}', \vec{\omega}')$, which accounts for attenuation along the direct path [S1].
2.  **Integration Setup:** For every point $\mathbf{x}''$ along the path from $\mathbf{x}$ to $\mathbf{x}'$ (parameterized by distance $s'$ from $\mathbf{x}'$ or $s-s'$ from $\mathbf{x}$), consider a scattering event [S1].
3.  **First Attenuation:** Light travels from the start $\mathbf{x}$ to the intermediate point $\mathbf{x}''$ [S1]. This is modeled by $G_{noscatter}(s - s', \mathbf{x}, \vec{\omega}, \mathbf{x}'', \vec{\omega}'')$, where $\vec{\omega}''$ is the direction from $\mathbf{x}$ to $\mathbf{x}''$ [S1]. This term includes attenuation over the distance $s - s'$ [S1].
4.  **Scattering:** At $\mathbf{x}''$, light scatters from direction $\vec{\omega}''$ to a new direction $\vec{\omega}'''$ [S1]. The probability of this scattering is determined by the product of the scattering coefficient $b(\mathbf{x}'')$ and the phase function $P(\mathbf{x}'', \vec{\omega}'', \vec{\omega}''')$ [S1].
5.  **Second Attenuation:** The scattered light travels from $\mathbf{x}''$ to the final point $\mathbf{x}'$ [S1]. This is modeled by $G_{noscatter}(s', \mathbf{x}'', \vec{\omega}''', \mathbf{x}', \vec{\omega}')$, including attenuation over the remaining distance $s'$ [S1].
6.  **Summation:** Integrate the product of these terms over all possible intermediate positions $\mathbf{x}''$ and all possible outgoing directions $\vec{\omega}'''$ (and incoming directions $\vec{\omega}''$ implied by geometry) along the path [S1].

**Formula:**
The total single scattering propagator is the sum of the no-scatter term and the single-scatter integral [S1]:

$$
G_{single}(s, \mathbf{x}, \vec{\omega}, \mathbf{x}', \vec{\omega}') = G_{noscatter}(s, \mathbf{x}, \vec{\omega}, \mathbf{x}', \vec{\omega}') + \int_0^s ds' \int_{\mathbf{x}''} \int_{\vec{\omega}''} \int_{\vec{\omega}'''} G_{noscatter}(s - s', \mathbf{x}, \vec{\omega}, \mathbf{x}'', \vec{\omega}'') \cdot b(\mathbf{x}'') \cdot P(\mathbf{x}'', \vec{\omega}'', \vec{\omega}''') \cdot G_{noscatter}(s', \mathbf{x}'', \vec{\omega}''', \mathbf{x}', \vec{\omega}') \, d\vec{\omega}'' \, d\vec{\omega}''' \, d\mathbf{x}''
$$

Where:
*   $s$ is the total path length [S1].
*   $s'$ is the distance from the intermediate point $\mathbf{x}''$ to the final point $\mathbf{x}'$ [S1].
*   $\mathbf{x}''$ is the intermediate scattering position [S1].
*   $\vec{\omega}''$ is the direction from $\mathbf{x}$ to $\mathbf{x}''$ [S1].
*   $\vec{\omega}'''$ is the direction from $\mathbf{x}''$ to $\mathbf{x}'$ [S1].
*   $b(\mathbf{x}'')$ is the scattering coefficient at $\mathbf{x}''$ [S1].
*   $P(\mathbf{x}'', \vec{\omega}'', \vec{\omega}''')$ is the phase function [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_single_scattering_propagator(x_view, omega_view, x_light, s_total, sigma_t_func, b_func, P_func, G_noscatter_func):
    """
    Computes G_single via ray marching, discretizing the spatial integral.
    Corresponds to the standard single scattering ray marching algorithm [S1].
    """
    # 1. Direct transmission component (no scattering)
    direct_term = G_noscatter_func(s_total, x_view, omega_view, x_view + omega_view * s_total, omega_view)
    
    # 2. Single scattering component
    single_scatter_term = 0.0
    num_steps = 100  # Discretization resolution [G]
    ds = s_total / num_steps
    
    for i in range(1, num_steps + 1):
        s_to_inter = i * ds
        s_from_inter = s_total - s_to_inter
        
        x_double_prime = x_view + omega_view * s_to_inter
        omega_double_prime = omega_view
        
        # Shadow ray direction towards light source
        dir_to_light = x_light - x_double_prime
        dist_to_light = max(norm(dir_to_light), 1e-6)
        omega_triple_prime = dir_to_light / dist_to_light
        
        # First attenuation: view ray from x_view to x''
        atten1 = G_noscatter_func(s_to_inter, x_view, omega_double_prime, x_double_prime, omega_double_prime)
        
        # Scattering coefficient and phase function
        b_val = b_func(x_double_prime)
        phase_val = P_func(x_double_prime, omega_double_prime, omega_triple_prime)
        
        # Second attenuation: shadow ray from x'' to light source
        atten2 = G_noscatter_func(dist_to_light, x_double_prime, omega_triple_prime, x_light, omega_triple_prime)
        
        single_scatter_term += atten1 * b_val * phase_val * atten2 * ds
    
    return direct_term + single_scatter_term
```
*Note: For the general propagator definition, the angular integral is approximated in practice by sampling discrete light source directions during ray marching [S1].*

**Parameters:**
*   $G_d$ is computed using existing techniques for direct lighting [S1].
*   $G_s$ (multiply scattered) is the primary target for efficient rendering algorithms, implying $G_d$ is often considered "solved" or handled by standard rasterization/ray tracing [S1].

**Edge Cases and Pitfalls:**
*   **Complexity:** Recursively expanding the propagator for higher orders of scattering becomes unmanageably complex due to additional angular integrations [S1].
*   **Initial Condition:** The scattered light propagator $G_s$ starts at zero: $G_s(s=0, \mathbf{x}, \vec{\omega}, \mathbf{x}', \vec{\omega}') = 0$ [S1].

## Subtopics

### [Utah Approximation (Single Scattering)](1717-utah-approximation-single-scattering.md)
A shading model for single scattering where external illumination reaches a voxel from a distant source unimpeded by intervening objects or volume absorption[1717-utah-approximation-single-scattering.md]. It differs from the Single Scattering Propagator by ignoring occlusion and attenuation between the light source and the voxel, serving as the foundational step for adding surface-like shading to volume data [1717-utah-approximation-single-scattering.md]. Choose it when performance is critical and the medium is optically thin or when surface-like shading is sufficient; use the propagator when rigorous attenuation and phase function integration are required.

## Operating characteristics
*   **Time Complexity:** The direct evaluation of the integral is computationally expensive due to the multiple integrals over space and angles [S1]. The corresponding ray marching algorithm is $O(N \cdot L)$ where $N$ is the number of steps along the view ray and $L$ is the number of light sources (shadow rays) [S1].
*   **Memory Complexity:** Low, as it typically processes one ray or voxel at a time without storing large intermediate scattering fields for the direct component [G].
*   **Strengths:** Physically accurate for thin media or when direct lighting dominates [S1]. It is the standard baseline for single scattering [S1].
*   **Limitations:** Does not account for indirect lighting (multiple scattering) [S1]. Becoming unmanageably complex if extended to higher orders [S1].
*   **When to use:** When the medium is optically thin or when direct lighting is the primary visual cue [S1].
*   **When not to use:** In dense media where multiple scattering is significant; use $G_s$ methods instead [S1].

## Background
The Single Scattering Propagator formalizes the concept of light scattering exactly once, separating it from the unscattered (direct) and multiply scattered (indirect) components [S1]. This separation is crucial for efficient rendering algorithms, as $G_d$ can be handled by existing direct lighting techniques, while $G_s$ requires specialized methods for participating media [S1]. The formulation corresponds directly to the standard single scattering ray marching algorithm used in computer graphics [S1]. The path integral approach provides an alternative way to express the full propagator $G$ as a sum over transfer events along different paths [S1].

## Sources
[S1] PDF Practical Rendering of Multiple Scattering Effects in Participating Media, https://cseweb.ucsd.edu/~ravir/HRPIEG.pdf, fetched 2026-10-28
