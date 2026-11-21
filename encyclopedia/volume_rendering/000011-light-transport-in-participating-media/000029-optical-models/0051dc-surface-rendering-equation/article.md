# Surface Rendering Equation

Also called: Boundary Condition for Volume Rendering Equation

## Overview
The Surface Rendering Equation defines the outgoing radiance at a surface point as the sum of emitted radiance and reflected incoming radiance, serving as the boundary condition for the Volume Rendering Equation [S1, unofficial]. It is distinct from volume-based models by restricting light transport interactions to the 2D manifold of opaque surfaces rather than the 3D participating medium volume [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** Surface position $\mathbf{x}_S$, outgoing view direction $\boldsymbol{\omega}$, surface normal $\vec{n}$, incoming radiance field $L_i(\mathbf{x}_S, \boldsymbol{\omega}')$, surface emission $L_e^S$, and surface scattering function $f_s$.
*   **Output:** Outgoing radiance $L_o(\mathbf{x}_S, \boldsymbol{\omega})$.

**Data Structures**
No specific data structures are required beyond standard vector and scalar types for position, direction, and radiance values. The surface normal $\vec{n}$ is a unit vector perpendicular to the surface at $\mathbf{x}_S$.

**Steps**
1.  **Determine Surface Intersection:** Identify the surface point $\mathbf{x}_S$ corresponding to the ray origin $\mathbf{x}$ and direction $-\boldsymbol{\omega}$ in the medium. This is achieved via a raycast function:
    $$ \mathbf{x}_S = \text{raycast}(\mathbf{x}, -\boldsymbol{\omega}) $$
    where $\text{raycast}$ returns the nearest object surface point along the specified direction [S1, unofficial].
2.  **Evaluate Surface Emission:** Retrieve the surface emission radiance $L_e^S(\mathbf{x}_S, \boldsymbol{\omega})$ at the intersection point for the outgoing direction [S1, unofficial].
3.  **Integrate Reflected Radiance:** Compute the integral of incoming radiance over the hemisphere $\mathcal{S}^2$ above the surface. For each incoming direction $\boldsymbol{\omega}'$, sample the incoming radiance $L_i(\mathbf{x}_S, \boldsymbol{\omega}')$, multiply by the surface bidirectional scattering distribution function $f_s(\mathbf{x}_S, \boldsymbol{\omega}', \boldsymbol{\omega})$, and weight by the cosine of the angle between the incoming direction and the surface normal $|\boldsymbol{\omega}' \cdot \vec{n}|$ [S1, unofficial].
4.  **Sum Components:** Add the surface emission and the integrated reflected radiance to obtain the total outgoing radiance [S1, unofficial].

**Formula**
$$ L_o(\mathbf{x}_S, \boldsymbol{\omega}) = L_e^S(\mathbf{x}_S, \boldsymbol{\omega}) + \int_{\mathcal{S}^2} f_s(\mathbf{x}_S, \boldsymbol{\omega}', \boldsymbol{\omega}) L_i(\mathbf{x}_S, \boldsymbol{\omega}') |\boldsymbol{\omega}' \cdot \vec{n}| \, d\boldsymbol{\omega}' $$

**Symbol Definitions**
*   $L_o(\mathbf{x}_S, \boldsymbol{\omega})$: Outgoing radiance at surface point $\mathbf{x}_S$ in direction $\boldsymbol{\omega}$.
*   $L_e^S(\mathbf{x}_S, \boldsymbol{\omega})$: Surface emission radiance at $\mathbf{x}_S$ in direction $\boldsymbol{\omega}$ [S1, unofficial].
*   $f_s(\mathbf{x}_S, \boldsymbol{\omega}', \boldsymbol{\omega})$: Surface bidirectional scattering distribution function (BSDF) at $\mathbf{x}_S$, describing scattering from incoming direction $\boldsymbol{\omega}'$ to outgoing direction $\boldsymbol{\omega}$ [S1, unofficial].
*   $L_i(\mathbf{x}_S, \boldsymbol{\omega}')$: Incoming radiance at $\mathbf{x}_S$ from direction $\boldsymbol{\omega}'$ [S1, unofficial].
*   $\vec{n}$: Surface normal at $\mathbf{x}_S$ [S1, unofficial].
*   $\mathcal{S}^2$: The unit hemisphere of directions above the surface [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def surface_rendering_equation(x_S, omega, n, L_e_S, f_s, get_incoming_radiance):
    """
    Computes outgoing radiance at a surface point.
    
    Args:
        x_S: Surface position vector.
        omega: Outgoing view direction vector.
        n: Surface normal vector.
        L_e_S: Function returning surface emission radiance L_e^S(x_S, omega).
        f_s: Function returning BSDF f_s(x_S, omega_prime, omega).
        get_incoming_radiance: Function returning L_i(x_S, omega_prime).
        
    Returns:
        L_o: Outgoing radiance.
    """
    # Step 1: Get surface emission
    emission = L_e_S(x_S, omega)
    
    # Step 2: Integrate reflected radiance over the hemisphere
    # Note: In practice, this integral is approximated using Monte Carlo integration
    # or other numerical methods. The exact integration is not given in sources.
    reflected_radiance = 0.0
    
    # Placeholder for numerical integration over hemisphere S^2
    # For each sample direction omega_prime in hemisphere:
    #   weight = |dot(omega_prime, n)|
    #   L_i = get_incoming_radiance(x_S, omega_prime)
    #   bsdf = f_s(x_S, omega_prime, omega)
    #   reflected_radiance += bsdf * L_i * weight
    # reflected_radiance /= number_of_samples (if using MC)
    
    # Step 3: Sum emission and reflection
    L_o = emission + reflected_radiance
    
    return L_o
```

**Edge Cases and Pitfalls**
*   **Shadowing:** The formula assumes all incoming directions $\boldsymbol{\omega}'$ are visible. In practice, visibility functions (shadow rays) must be included to account for occlusion, though this is not explicitly detailed in the source formula [G].
*   **Degenerate Normals:** If $\vec{n}$ is undefined or zero, the cosine term $|\boldsymbol{\omega}' \cdot \vec{n}|$ becomes zero, resulting in no reflected light.
*   **Precision:** Radiance values can vary widely; using appropriate data types (e.g., float32 or float64) is critical to avoid clipping or underflow.

## Subtopics

### [Bidirectional Reflectance Distribution Function (BRDF)](14809-bidirectional-reflectance-distribution-function-brdf.md)
The BRDF is a specific case of the BSDF where the surface is opaque and does not transmit light. It is chosen when modeling reflection-only surfaces.

### [Bidirectional Transmittance Distribution Function (BTDF)](52-btdf.md)
Models light transmission through a surface. It is chosen when the surface allows light to pass through, such as glass or thin films.

### [Bidirectional Scattering Surface Reflectance Distribution Function (BSSRDF)](52-bssrdf.md)
Models subsurface scattering where light penetrates the surface volume. It is chosen for materials like skin, wax, or marble where light diffuses within the material before exiting.

## Operating characteristics
*   **Time Complexity:** The integral over the hemisphere $\mathcal{S}^2$ typically requires numerical integration (e.g., Monte Carlo), making the cost proportional to the number of samples used to approximate the integral.
*   **Memory Complexity:** Minimal, requiring storage for surface properties (BSDF, emission) and normal vectors.
*   **Strengths:** Provides a physically accurate boundary condition for volume rendering, allowing seamless integration of surface and volume effects.
*   **Limitations:** Requires accurate surface intersection and normal estimation. Computationally expensive if high-quality integration of the BSDF is needed.
*   **When to Use:** When rendering scenes with both participating media and opaque/translucent surfaces, and when accurate surface-light interactions are required.
*   **When Not to Use:** For purely volumetric effects without surface boundaries, or when performance constraints prohibit complex surface shading calculations.

## Background
The Surface Rendering Equation is derived from the principles of radiative transfer and serves as the boundary condition for the Volume Rendering Equation [S1, unofficial]. It was formalized in the context of volume rendering to handle the interaction between light transport in participating media and opaque surfaces [S1, unofficial]. The concept builds upon the foundational work in computer graphics regarding surface shading, extending it to be compatible with volumetric light transport.

## Sources
[S1] ボリュームレンダリング方程式(Volume Rendering Equation) 2, https://rayspace.xyz/notes/VLTE2/, fetched 2026-10-30 (unofficial)
