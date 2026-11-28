# Photon Mapping for RRTE

Also called: Extended Photon Mapping, RRTE-based photon mapping, Refractive radiative transfer equation-based photon mapping

## Overview
Photon Mapping for the Refractive Radiative Transfer Equation (RRTE) is a two-pass rendering algorithm designed to simulate light transport in participating media with spatially varying refractive indices [S1]. It solves the integral form of the RRTE using basic radiance to ensure energy conservation, addressing the limitation of standard photon mapping which assumes a constant index of refraction [S1]. Unlike standard photon mapping, this method computes curved photon and ray trajectories analytically or numerically to model continuous refraction, producing altered caustic shapes and more accurate reflections [S1].

## How it works
The method consists of two passes: Photon Tracing and Radiance Estimation. It handles both steady-state and transient scenarios.

### Inputs and Outputs
**Inputs:**
*   Light source definitions (position, direction, flux).
*   Medium properties: extinction coefficient $\sigma_t(\mathbf{x})$, scattering coefficient $\sigma_s(\mathbf{x})$, phase function $p(\boldsymbol{\omega}, \boldsymbol{\omega}')$, and spatially varying index of refraction $n(\mathbf{x})$ [S1].
*   Geometry of the scene [S1].
*   For transient mode: Time step $\Delta t$ and total simulation time [S1].

**Outputs:**
*   A photon map storing virtual photons with their positions, directions, flux, and (for transient mode) Time of Flight (TOF) [S1].
*   Final rendered image with physically correct global illumination [S1].

### Data Structures
**Photon Map Entry:**
Each stored photon contains:
*   Position $\mathbf{x}$ [S1].
*   Direction $\boldsymbol{\omega}$ [S1].
*   Flux $W$ (or weight) [S1].
*   Radius $r$ (for beam radiance estimation) [S1].
*   Bounding box of the photon disc (for acceleration structure) [S1].
*   *Transient only:* Time of Flight (TOF) [S1].
*   *Transient only:* Unique light source identifier [S1].

**Acceleration Structure:**
*   A bounding box hierarchy (e.g., BVH) built over the photon discs to accelerate density estimation during the second pass [S1].

### Formulas
**Index of Refraction Models:**
1.  **Linear vertical profile:**
    $$n(y) = (1.0 - y) \cdot n_{bottom} + y \cdot n_{top}$$
    where $y \in [0, 1]$ is the normalized vertical axis position [S1].

2.  **Heterogeneous material mix:**
    $$n(\mathbf{x}) = C_g(\mathbf{x}) \cdot n_g + C_j(\mathbf{x}) \cdot n_j$$
    where $C_g(\mathbf{x}) \in [0, 1]$ is the concentration of material $g$, $C_j(\mathbf{x}) = 1.0 - C_g(\mathbf{x})$ is the concentration of material $j$, and $n_g, n_j$ are their respective refractive indices [S1].

**Group Velocity (Transient):**
$$v_g(\mathbf{x}) = \frac{c}{n(\mathbf{x})}$$
where $c$ is the speed of light in vacuum [S1].

**Spatial Step Size (Transient):**
$$\Delta s(\mathbf{x}) = v_g(\mathbf{x}) \Delta t$$
where $\Delta t$ is the time step size [S1].

### Parameters
*   $\Delta t$: Step size for numerical integration in transient photon tracing [S1].
*   Photon radius: Assigned to each photon for beam radiance estimation [S1].
*   Number of photons $N$: Determines the weight of each emitted photon as $1/N$ [S1].

### Steps

#### Pass 1: Photon Tracing
Virtual photons are emitted from light sources and traced through the medium.

1.  **Emit Photon:** For each light source, emit a photon with initial weight $W = 1/N$ and a unique source ID (if transient) [S1].
2.  **Trace Trajectory:** Compute the curved path $\mathbf{x}(s)$ through the medium with varying $n(\mathbf{x})$ [S1].
    *   **Steady-State:** Use the fourth-order Runge-Kutta (RK4) method to solve the ray equation [S1].
    *   **Transient:** Solve for position $\mathbf{x}(t)$ and velocity $\mathbf{v}(t)$ numerically using step size $\Delta s(\mathbf{x}) = v_g(\mathbf{x})\Delta t$. Accumulate TOF by adding $\Delta t$ at each step [S1].
3.  **Interaction Sampling:** Determine if the photon interacts with the medium using the inversion method or Woodcock tracking based on $\sigma_t(\mathbf{x})$ [S1].
4.  **Store Photon:** If an interaction occurs, store the photon in the photon map.
    *   **Scattering Handling:** Store photons after the **first bounce** (single scattering) as well as subsequent bounces. This is necessary because curved trajectories prevent general shadow ray tracing to sample direct lighting [S1].
5.  **Russian Roulette:** Use Russian roulette to decide if the photon is absorbed or scattered. If scattered, update direction $\boldsymbol{\omega}$ based on the phase function and reduce weight $W$ by the albedo [S1].
6.  **Repeat:** Continue tracing until the photon is absorbed or escapes the scene [S1].

#### Pass 2: Radiance Estimation
Ray march from the camera to solve the integral form of the transfer equation.

1.  **Ray Marching:** For each pixel, cast a ray into the scene. Sample points along the ray [S1].
2.  **Density Estimation:** At each sample point, query the photon map to find nearby photons within the assigned radius [S1].
3.  **Radiance Calculation:** Estimate the in-scattered radiance using the stored photon fluxes and the beam radiance estimation method [S1].
4.  **Accumulate:** Accumulate the estimated radiance along the ray, accounting for extinction [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def trace_steady_state_photon(light_source, n_field, sigma_t, sigma_s, phase_func, photon_map, rk4_step_size):
    """
    Trace a single photon in steady-state mode.
    """
    # Initialize photon
    pos = light_source.position
    dir = light_source.direction
    flux = 1.0 / light_source.num_photons
    current_weight = flux
    
    # Initial step
    s = 0.0
    
    while current_weight > 0.01: # Threshold for Russian Roulette
        # 1. Determine free path distance (Woodcock or Inversion)
        # Not given in sources: Implementation of free path sampling
        delta_s = sample_free_path(pos, sigma_t)
        
        # 2. Move photon along curved trajectory
        # Use RK4 to integrate ray equation over delta_s
        # Not given in sources: Full RK4 integration of d/ds(n * dx/ds) = grad(n)
        new_pos = rk4_integrate(pos, dir, n_field, delta_s, rk4_step_size)
        
        # 3. Check for interaction
        # Not given in sources: Check if sampled distance < delta_s
        if interacted:
            # Store photon (including first bounce)
            photon = Photon(new_pos, dir, current_weight, radius=0.0) # Radius assigned later
            photon_map.insert(photon)
            
            # Russian Roulette
            if random() < sigma_s / sigma_t:
                # Scatter
                dir = sample_phase_function(dir, phase_func)
                current_weight *= sigma_s / sigma_t
                pos = new_pos
            else:
                # Absorbed
                break
        else:
            pos = new_pos
            
    return photon_map

def trace_transient_photon(light_source, n_field, sigma_t, sigma_s, phase_func, photon_map, delta_t):
    """
    Trace a single photon in transient mode.
    """
    c = 299792458.0 # Speed of light
    
    pos = light_source.position
    dir = light_source.direction
    flux = 1.0 / light_source.num_photons
    current_weight = flux
    tof = 0.0
    
    while current_weight > 0.01:
        # 1. Determine free path distance
        # Not given in sources: Implementation of free path sampling
        delta_s = sample_free_path(pos, sigma_t)
        
        # 2. Compute spatial step based on group velocity
        n_val = n_field.evaluate(pos)
        v_g = c / n_val
        delta_s_actual = v_g * delta_t
        
        # Use the smaller of the two steps
        step = min(delta_s, delta_s_actual)
        
        # 3. Move photon
        # Not given in sources: Numerical integration of x(t) and v(t)
        new_pos = integrate_transient(pos, dir, n_field, step)
        
        # 4. Update TOF
        tof += delta_t
        
        # 5. Check interaction
        if interacted:
            photon = Photon(new_pos, dir, current_weight, radius=0.0, tof=tof, source_id=light_source.id)
            photon_map.insert(photon)
            
            # Russian Roulette
            if random() < sigma_s / sigma_t:
                dir = sample_phase_function(dir, phase_func)
                current_weight *= sigma_s / sigma_t
                pos = new_pos
            else:
                break
        else:
            pos = new_pos
            
    return photon_map

def estimate_radiance(camera_ray, photon_map, radius):
    """
    Estimate radiance at a point along the camera ray using photon map.
    """
    radiance = 0.0
    
    # Find nearby photons within radius
    nearby_photons = photon_map.query_ball_point(camera_ray.position, radius)
    
    for photon in nearby_photons:
        # Beam radiance estimation
        # Not given in sources: Exact kernel function for beam radiance estimation
        contribution = estimate_contribution(photon, camera_ray.direction)
        radiance += contribution
        
    return radiance
```

### Edge Cases and Pitfalls
*   **Curved Trajectories:** Curved paths prevent the use of shadow rays to connect arbitrary points, making path tracing slow due to lack of direct lighting sampling. Photon mapping is chosen to handle this [S1].
*   **Single Scattering Storage:** Storing photons after the first bounce increases the necessary amount of photons to obtain high-quality results compared to non-refractive media [S1].
*   **Concentration Profile Errors:** Discrepancies between rendered and real-world results occur if the measured concentration profile deviates from the assumed distribution, causing blurring or trajectory errors near bounces [S1].
*   **Precision:** Numerical integration of curved trajectories requires careful handling of precision, especially with steep refractive index gradients [S1].

## Operating characteristics
*   **Time Complexity:** Render time increases due to the more expensive computation of curved trajectories and intersection points with geometry [S1].
    *   Measured increases: Cocktail scene 2:21 h to 3:16 h; Sphere scene 1:26 h to 2:19 h; Fluid sim cup scene ~3 h to 4:41 h [S1].
*   **Memory Complexity:** Requires storing photons with additional data (radius, bounding box, TOF, source ID) and building a bounding box hierarchy over photon discs [S1].
*   **Strengths:** Produces noticeably different global illumination, altered caustic shapes/positions, and more accurate reflections due to curved trajectories compared to standard photon mapping [S1]. Conserves energy using basic radiance [S1].
*   **Limitations:** Higher computational cost. Requires more photons for high-quality results due to single scattering storage. Sensitive to inaccuracies in the refractive index profile [S1].
*   **When to use:** When simulating participating media with spatially varying refractive indices where accurate caustics and refraction effects are critical [S1].
*   **When not to use:** When performance is critical and constant refractive index is a sufficient approximation [S1].

## Background
Photon Mapping for RRTE was developed by Zhang et al. as an extension of standard photon mapping to handle the Refractive Radiative Transfer Equation [S1]. It builds on the work of Gutierrez et al. [2005] on bending light in participating media and Jarosz et al. [2008] on beam radiance estimation [S1]. The method was introduced to conserve energy in participating media with spatially varying refractive indices, a problem not fully addressed by previous approaches that did not account for transient effects or basic radiance [S1]. It is used in computer graphics for physically based rendering of complex fluids and heterogeneous materials [S1].

## Sources
[S1] Refractive radiative transfer equation, https://doi.org/10.1145/2557605, fetched 2026-10-31
