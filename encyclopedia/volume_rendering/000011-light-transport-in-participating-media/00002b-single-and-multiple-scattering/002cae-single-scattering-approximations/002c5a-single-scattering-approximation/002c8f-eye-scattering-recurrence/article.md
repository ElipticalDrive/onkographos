# Eye Scattering Recurrence

Also called: Single scattering toward viewer recurrence

## Overview
The Eye Scattering Recurrence computes the light exiting each particle in a volume toward the viewer, assuming single scattering events relative to the viewing direction. It solves the problem of determining visible brightness by combining the transmitted light from previous particles with the light scattered by the current particle toward the camera. It differs from its sibling, Multiple Forward Scattering, which computes light incident on particles from the light source; this method uses the precomputed incident light to determine scattering toward the viewer [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A sequence of $N$ particles $p_1, \dots, p_N$ along the view ray. For each particle $k$, the albedo $a_k$, extinction $\tau_k$, and the light incident on the particle $I_k$ (computed in a preceding forward scattering pass) are required. The view direction $\omega$ and light direction $l$ are constant for the ray.
*   **Output:** The exit radiance $E_k$ for each particle $k$, representing the light leaving particle $k$ toward the viewer.

**Data Structures**
The method operates on a linear array of particles. Each particle $k$ stores:
*   $a_k$: Albedo (dimensionless, $0 \le a_k \le 1$).
*   $\tau_k$: Extinction coefficient (inverse length).
*   $I_k$: Incident light intensity (radiance) at particle $k$, computed in the previous pass.
*   $T_k$: Transparency of particle $k$, derived from extinction.

**Formulas**
The recurrence relation for $1 \le k \le N$ is:

$$ E_k = S_k + T_k \cdot E_{k-1} $$

Where:
*   $E_k$ is the light exiting particle $k$ toward the viewer.
*   $S_k$ is the light scattered by particle $k$ toward the viewer.
*   $T_k$ is the transparency of particle $k$.
*   $E_{k-1}$ is the light exiting the previous particle $k-1$. For $k=1$, $E_0 = 0$ (or background radiance if applicable, though the source implies starting from zero for the particle contribution).

The scattered light $S_k$ is defined as:

$$ S_k = \frac{a_k \cdot \tau_k \cdot p(\omega, -l) \cdot I_k}{4\pi} $$

Where:
*   $a_k$ is the albedo of particle $k$.
*   $\tau_k$ is the extinction of particle $k$.
*   $p(\omega, -l)$ is the phase function evaluated for the view direction $\omega$ and the negative light direction $-l$ (scattering from light direction to view direction).
*   $I_k$ is the light incident on particle $k$.
*   $4\pi$ is the normalization factor for isotropic scattering integration.

**Transparency Calculation**
The transparency $T_k$ is typically calculated as $e^{-\tau_k \Delta d}$, where $\Delta d$ is the distance between particles or the particle thickness. The source notes $T_k$ as "transparency of particle $k$" without explicit formula, but standard volume rendering defines it via extinction.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def eye_scattering_recurrence(particles, view_direction, light_direction):
    """
    Computes light exiting each particle toward the viewer.
    
    Args:
        particles: List of particle objects/dicts with fields:
                   - albedo (a_k)
                   - extinction (tau_k)
                   - incident_light (I_k)
                   - thickness (delta_d) or transparency (T_k)
        view_direction: Vector omega
        light_direction: Vector l
        
    Returns:
        List of exit radiances E_k
    """
    N = len(particles)
    E = [0.0] * N
    
    # Precompute phase function value
    # p(omega, -l) determines scattering from light direction to view direction
    phase_val = phase_function(view_direction, -light_direction)
    
    # Initialize previous exit radiance
    # E_0 is 0 for the first particle's contribution from previous particles
    prev_E = 0.0
    
    for k in range(N):
        p = particles[k]
        
        # Calculate transparency T_k
        # If T_k is not directly provided, compute from extinction and thickness
        if 'transparency' in p:
            T_k = p['transparency']
        else:
            # Standard Beer-Lambert law
            T_k = exp(-p['extinction'] * p['thickness'])
            
        # Calculate scattered light S_k
        # S_k = (a_k * tau_k * p(omega, -l) * I_k) / 4pi
        a_k = p['albedo']
        tau_k = p['extinction']
        I_k = p['incident_light']
        
        S_k = (a_k * tau_k * phase_val * I_k) / (4.0 * pi)
        
        # Recurrence: E_k = S_k + T_k * E_{k-1}
        E_k = S_k + T_k * prev_E
        
        E[k] = E_k
        
        # Update previous exit radiance for next iteration
        prev_E = E_k
        
    return E
```

**Edge Cases and Pitfalls**
*   **Double Counting Phase Function:** A common pitfall is assuming the phase function is applied twice to the same light interaction because it appears in both the forward scattering pass (computing $I_k$) and the eye scattering pass (computing $S_k$). This is incorrect. In the forward pass, the phase function determines scattering from $p_{k-1}$ to $p_k$ in the light direction. In the eye scattering pass, it determines scattering from $p_k$ to the viewer. The incident light $I_k$ is stored and used directly, preventing double counting even if the viewpoint is opposite the light source [S1, unofficial].
*   **Boundary Conditions:** The recurrence starts with $E_0 = 0$. If there is background light, it should be added to the final $E_N$ or handled as $E_0$ depending on the specific implementation of the volume rendering integral. The source does not specify background handling, so $E_0=0$ is the base case for particle contributions.

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of particles along the ray. Each particle requires a constant number of arithmetic operations.
*   **Memory Complexity:** $O(N)$ to store the incident light $I_k$ from the previous pass and the output $E_k$.
*   **Strengths:** Efficiently computes single scattering toward the viewer. It is simple to implement and integrates well with precomputed incident light.
*   **Limitations:** Only accounts for single scattering toward the viewer. It does not account for multiple scattering events in the view direction (glow). It relies on the accuracy of the incident light $I_k$ computed in the previous pass.
*   **When to Use:** When single scattering approximation is valid (low albedo/density) and computational efficiency is prioritized over full multiple scattering accuracy.

## Background
The Eye Scattering Recurrence is based on the single scattering toward the viewer approximation described by Dobashi et al. [S1, unofficial]. It is part of the Real-Time Cloud Rendering technique, which combines multiple forward scattering (pre-process) with single scattering toward the viewer to achieve realistic cloud shading efficiently. The phase function used is critical for cloud shading, as clouds exhibit anisotropic scattering [S1, unofficial].

## Sources
[S1] Real-Time Cloud Rendering, https://www.cs.unc.edu/techreports/01-005.pdf, fetched 2026-10-28 (unofficial)
