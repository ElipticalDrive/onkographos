# Photon Marching

Also called: Discrete Photon Beam Generation

## Overview
Photon Marching is a photon shooting method that generates "discrete photon beams" by depositing photons at uniform intervals along rays originating from light sources [S1]. It solves the problem of estimating in-scattered radiance by replacing the stochastic sampling of scattering events with a deterministic Riemann sum integration along the ray path [S1]. Unlike standard photon tracing, which uses a Markov random-walk to determine a single propagation distance per photon, Photon Marching distributes the power of a single photon among multiple deposited photons at fixed steps [S1]. This approach is analogous to ray marching from the camera, but applied to light transport from the source [S1].

## How it works
**Inputs:**
*   Light source power $\Phi_b$ (power of the photon beam entering the medium) [S1].
*   Medium extinction coefficient $\sigma_t$ and scattering coefficient $\sigma_s$ [S1].
*   Uniform step size $\Delta t$ [S1].
*   Ray origin $\mathbf{x}_b$ and direction $\tilde{\boldsymbol{\omega}}_b$ [S1].
*   Random offset $\xi$ for aliasing avoidance [S1].

**Outputs:**
*   A set of deposited photons forming a "discrete photon beam" [S1].
*   Each photon has position $\mathbf{x}_t$ and power $\Phi_{p_b}$ [S1].

**Data Structures:**
*   **Discrete Photon Beam:** A list of photon deposits. Each deposit contains:
    *   Position $\mathbf{x}_t \in \mathbb{R}^3$
    *   Power $\Phi_{p_b} \in \mathbb{R}$
    *   No explicit bit-packing is defined in the sources; standard floating-point representations are implied [S1].

**Steps:**
1.  **Initialize:** Set the current ray origin $\mathbf{x}_b$ and direction $\tilde{\boldsymbol{\omega}}_b$ from the light source. Select a random offset $\xi$ for the current beam to avoid aliasing [S1].
2.  **Iterate Steps:** For each step index $k \in \{0, 1, \dots, N-1\}$:
    a. Calculate the path length $t_k = (\xi + k)\Delta t$ [S1].
    b. Calculate the position $\mathbf{x}_t = \mathbf{x}_b + t_k \tilde{\boldsymbol{\omega}}_b$ [S1].
    c. Calculate the transmittance from the ray origin to $\mathbf{x}_t$: $T(t_k) = e^{-\sigma_t t_k}$ [S1].
    d. Calculate the power of the deposited photon $\Phi_{p_b}$ using Equation 15 [S1]:
       $$ \Phi_{p_b} = \sigma_s e^{-\sigma_t t_k} \|\mathbf{x}_b - \mathbf{x}_t\| \Phi_b \Delta t $$
       Note: $\|\mathbf{x}_b - \mathbf{x}_t\|$ is equivalent to $t_k$ if $\tilde{\boldsymbol{\omega}}_b$ is a unit vector, simplifying the term to $\sigma_s e^{-\sigma_t t_k} t_k \Phi_b \Delta t$ [S1].
    e. Store the photon at $\mathbf{x}_t$ with power $\Phi_{p_b}$ [S1].
3.  **Terminate:** Stop when the ray exits the medium or reaches the maximum extent [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def photon_march(light_power, sigma_s, sigma_t, origin, direction, step_size, max_steps):
    """
    Generates a discrete photon beam using Photon Marching.
    
    Args:
        light_power (float): Phi_b, power of the photon beam entering the medium.
        sigma_s (float): Scattering coefficient.
        sigma_t (float): Total extinction coefficient.
        origin (Vector3): x_b, ray origin.
        direction (Vector3): omega_b, unit ray direction.
        step_size (float): Delta t, uniform step size.
        max_steps (int): N, maximum number of steps.
        
    Returns:
        list of dict: List of deposited photons with 'position' and 'power'.
    """
    # [G] Generate a random offset to avoid aliasing, as per source description.
    # The specific value is inconsequential for derivations but required for implementation.
    random_offset = random.uniform(0, step_size)
    
    deposited_photons = []
    
    for k in range(max_steps):
        # Calculate path length t
        t = (random_offset + k) * step_size
        
        # Calculate position x_t
        position = origin + direction * t
        
        # Calculate transmittance T = exp(-sigma_t * t)
        transmittance = math.exp(-sigma_t * t)
        
        # Calculate distance from origin (norm of vector)
        # Since direction is unit vector, this is simply t.
        distance = t 
        
        # Calculate photon power Phi_pb
        # Formula: Phi_pb = sigma_s * exp(-sigma_t * t) * ||x_b - x_t|| * Phi_b * Delta t
        photon_power = sigma_s * transmittance * distance * light_power * step_size
        
        # Store photon
        deposited_photons.append({
            'position': position,
            'power': photon_power
        })
        
        # [G] In practice, one would check if position is outside the medium volume.
        # If so, break loop. Not explicitly detailed in source steps, but implied by "extent of the beam".
        
    return deposited_photons
```

**Parameters:**
*   $\Delta t$: Uniform step size. Not given in the sources.
*   $\xi$: Random offset. Not given in the sources.
*   $N$: Total number of photons deposited. Depends on step-size and extent of the beam through the medium [S1].

**Edge Cases and Pitfalls:**
*   **Aliasing:** Using a fixed offset for all beams can cause aliasing. A random offset $\xi$ per beam is required [S1].
*   **Medium Extent:** The method assumes the medium has finite extent. If the medium is infinite, the number of steps $N$ is undefined [S1].
*   **Homogeneous vs Heterogeneous:** The derivation assumes uniform step size, implying a homogeneous medium or a piecewise approximation. For heterogeneous media, the step size or coefficients may need adjustment, though the source primarily describes the homogeneous case for the basic definition [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of steps $N$ and the length of the ray through the medium [S1].
*   **Memory Complexity:** Linear with respect to the number of deposited photons, which is proportional to $N$ [S1].
*   **Strengths:** Provides a deterministic integration of in-scattered light along the ray, analogous to ray marching. It avoids the variance associated with random-walk sampling for the initial beam generation [S1].
*   **Limitations:** It is a single-scattering approximation for the beam generation itself. Multiple scattering requires combining this with random-walk sampling (scattering a new discrete beam off the current beam) [S1].
*   **When to use:** When a deterministic integration of light transport from a light source is preferred over stochastic sampling for the initial beam, or when deriving mathematical definitions of photon beams [S1].

## Background
Photon Marching was introduced in the context of a comprehensive theory of volumetric radiance estimation using photon points and beams [S1]. It is described as a method to derive a mathematical definition of photon beams, although in practice, photon beams are still often created using a random-walk process [S1]. The method distinguishes itself from standard photon tracing by distributing photon power among multiple deposits rather than a single random-walk step [S1].

## Sources
[S1] A comprehensive theory of volumetric radiance estimation using photon points and beams, https://doi.org/10.1145/1899404.1899409, fetched 2026-10-26
