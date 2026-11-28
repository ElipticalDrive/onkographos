# Surface Rendering Equation

Also called: Boundary Condition, Boundary Condition for Volume Rendering Equation, Boundary Conditions for Transfer Equation, Kajiya's Rendering Equation, Reflectance Equation, Rendering Equation (Vacuum Case)

## Overview
The Surface Rendering Equation defines the outgoing radiance at a surface point as the sum of emitted radiance and reflected incoming radiance, serving as the boundary condition for the Volume Rendering Equation [S2, S3, unofficial]. It restricts light transport interactions to the 2D manifold of surfaces rather than the 3D participating medium volume, reducing to the standard vacuum transport equation when no medium is present [S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** Surface position $\mathbf{x}_S$, outgoing view direction $\boldsymbol{\omega}$, surface normal $\vec{n}$, incoming radiance field $L_i(\mathbf{x}_S, \boldsymbol{\omega}')$, surface emission $L_e^S$, and surface scattering function $f_s$ [S3, unofficial].
*   **Output:** Outgoing radiance $L_o(\mathbf{x}_S, \boldsymbol{\omega})$ [S3, unofficial].

**Data Structures**
Standard vector and scalar types for position, direction, and radiance values are sufficient [G].

**Steps**
1.  **Determine Surface Intersection:** Identify the surface point $\mathbf{x}_S$ corresponding to the ray origin $\mathbf{x}$ and direction $-\boldsymbol{\omega}$ in the medium using a raycast function [S3, unofficial].
2.  **Evaluate Surface Emission:** Retrieve the surface emission radiance $L_e^S(\mathbf{x}_S, \boldsymbol{\omega})$ at the intersection point for the outgoing direction [S3, unofficial].
3.  **Integrate Reflected Radiance:** Compute the integral of incoming radiance over the unit sphere of directions $\mathcal{S}^2$, weighting by the surface bidirectional scattering distribution function $f_s$ and the cosine of the angle between the incoming direction and the surface normal [S3, unofficial].
4.  **Sum Components:** Add the surface emission and the integrated reflected radiance to obtain the total outgoing radiance [S3, unofficial].

**Formula**
$$ L_o(\mathbf{x}_S, \boldsymbol{\omega}) = L_e^S(\mathbf{x}_S, \boldsymbol{\omega}) + \int_{\mathcal{S}^2} f_s(\mathbf{x}_S, \boldsymbol{\omega}', \boldsymbol{\omega}) L_i(\mathbf{x}_S, \boldsymbol{\omega}') |\boldsymbol{\omega}' \cdot \vec{n}| \, d\boldsymbol{\omega}' $$

**Symbol Definitions**
*   $L_o(\mathbf{x}_S, \boldsymbol{\omega})$: Outgoing radiance at surface point $\mathbf{x}_S$ in direction $\boldsymbol{\omega}$ [S3, unofficial].
*   $L_e^S(\mathbf{x}_S, \boldsymbol{\omega})$: Surface emission radiance at $\mathbf{x}_S$ in direction $\boldsymbol{\omega}$ [S3, unofficial].
*   $f_s(\mathbf{x}_S, \boldsymbol{\omega}', \boldsymbol{\omega})$: Surface bidirectional scattering distribution function (BSDF) at $\mathbf{x}_S$, describing scattering from incoming direction $\boldsymbol{\omega}'$ to outgoing direction $\boldsymbol{\omega}$ [S3, unofficial].
*   $L_i(\mathbf{x}_S, \boldsymbol{\omega}')$: Incoming radiance at $\mathbf{x}_S$ from direction $\boldsymbol{\omega}'$ [S3, unofficial].
*   $\vec{n}$: Surface normal at $\mathbf{x}_S$ [S3, unofficial].
*   $\mathcal{S}^2$: The unit sphere of all directions; the cosine term $|\boldsymbol{\omega}' \cdot \vec{n}|$ restricts contributions to the hemisphere above the surface [S3, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def surface_rendering_equation(x_S, omega, n, L_e_S, f_s, get_incoming_radiance, num_samples=1000):
    """
    Computes outgoing radiance at a surface point using Monte Carlo integration.
    
    Args:
        x_S: Surface position vector.
        omega: Outgoing view direction vector.
        n: Surface normal vector.
        L_e_S: Function returning surface emission radiance L_e^S(x_S, omega).
        f_s: Function returning BSDF f_s(x_S, omega_prime, omega).
        get_incoming_radiance: Function returning L_i(x_S, omega_prime).
        num_samples: Number of Monte Carlo samples for integration.
        
    Returns:
        L_o: Outgoing radiance.
    """
    # Step 1: Get surface emission
    emission = L_e_S(x_S, omega)
    
    # Step 2: Integrate reflected radiance over the hemisphere
    # The sources do not specify a numerical integration method, so standard Monte Carlo is used [G].
    reflected_radiance = 0.0
    for _ in range(num_samples):
        # Sample a direction uniformly over the hemisphere defined by n [G]
        omega_prime = sample_uniform_hemisphere(n)
        # Cosine weighting factor
        weight = abs(dot(omega_prime, n))
        # Sample incoming radiance and BSDF
        L_i = get_incoming_radiance(x_S, omega_prime)
        bsdf = f_s(x_S, omega_prime, omega)
        # Accumulate estimator
        reflected_radiance += bsdf * L_i * weight
    # Average over samples
    reflected_radiance /= num_samples
    
    # Step 3: Sum emission and reflection
    L_o = emission + reflected_radiance
    
    return L_o

def sample_uniform_hemisphere(n):
    """Generates a random unit vector uniformly distributed over the hemisphere defined by normal n [G]."""
    # Standard spherical sampling adjusted to local hemisphere frame [G]
    u = random()
    v = random()
    theta = 2.0 * pi * u
    phi = acos(1.0 - 2.0 * v)
    # Convert to local frame and rotate to align with n
    local_dir = vec3(sin(phi) * cos(theta), sin(phi) * sin(theta), cos(phi))
    return rotate_to_normal(local_dir, n)
```

**Edge Cases and Pitfalls**
*   **Shadowing:** The formula assumes all incoming directions $\boldsymbol{\omega}'$ are visible; in practice, visibility functions or shadow rays must be included to account for occlusion [G].
*   **Degenerate Normals:** If $\vec{n}$ is undefined or zero, the cosine term $|\boldsymbol{\omega}' \cdot \vec{n}|$ becomes zero, resulting in no reflected light [G].
*   **Precision:** Radiance values can vary widely; using appropriate data types (e.g., float32 or float64) is critical to avoid clipping or underflow [G].

## Subtopics

### [Bidirectional Reflectance Distribution Function (BRDF)](14809-bidirectional-reflectance-distribution-function-brdf.md)
The BRDF provides the ratio of flux concentration per steradian reflected into a given direction as a consequence of differential irradiance incident from another given direction. It serves as the core component of the Surface Rendering Equation, defining the reflected radiance at a surface point.

## Operating characteristics
*   **Time Complexity:** The integral over $\mathcal{S}^2$ typically requires numerical integration (e.g., Monte Carlo), making the cost proportional to the number of samples used to approximate the integral [G].
*   **Memory Complexity:** Minimal, requiring storage for surface properties (BSDF, emission) and normal vectors [G].
*   **Strengths:** Provides a physically accurate boundary condition for volume rendering, allowing seamless integration of surface and volume effects [S2, unofficial].
*   **Limitations:** Requires accurate surface intersection and normal estimation, and becomes computationally expensive if high-quality integration of the BSDF is needed [G].
*   **When to Use:** When rendering scenes with both participating media and opaque/translucent surfaces, and when accurate surface-light interactions are required [G].
*   **When Not to Use:** For purely volumetric effects without surface boundaries, or when performance constraints prohibit complex surface shading calculations [G].

## Background
The concept was introduced by Kajiya (1986) and Immel, Cohen, and Greenberg (1986) [S1, unofficial]. It serves as the explicit boundary condition for the radiative transfer equation at surface $S$, where the surface normal $\mathbf{N}$ points into the volume [S2, unofficial]. In the absence of a participating medium, the transport equation reduces to constant intensity along rays, and surface intensity is determined entirely by this boundary condition [S2, unofficial].

## Sources
[S1] A Tile-Based Approach for Photo-Realistic Volume Rendering, http://hdl.handle.net/1794/23836, fetched 2026-10-26 (unofficial)
[S2] Volume Rendering Mathematical Models and Algorithmic Aspects, https://core.ac.uk/works/16970577, fetched 2026-10-29 (unofficial)
[S3] ボリュームレンダリング方程式(Volume Rendering Equation) 2, https://rayspace.xyz/notes/VLTE2/, fetched 2026-10-30 (unofficial)
