# Grid-based Convection-Diffusion Illumination

Also called: Convection-diffusion equation method, PDE-based volume illumination

## Overview
Grid-based Convection-Diffusion Illumination is a real-time volume rendering technique that models light propagation, absorption, and multiple scattering within participating media by solving a unified partial differential equation (PDE) on a regular grid [S1]. It distinguishes itself from Monte Carlo ray tracing by separating illumination modeling from simulation, allowing the use of highly parallelized numerical schemes for real-time performance, and from Summed Area Table (SAT) techniques by producing realistic multiple scattering effects rather than just global shadows [S1].

## How it works
The method computes a "light volume" representing the energy density distribution inside the participating medium, which is then used during standard volume ray casting to shade sampling points [S1]. The computation is divided into two main phases: direct illumination (convection) and multiple scattering (diffusion) [S1].

### Inputs and Outputs
**Inputs:**
*   Volume dataset $V$ with spatially varying absorption coefficient $\sigma_a(x)$ and scattering coefficient $\sigma_s$ [S1].
*   A set of $N$ light sources. For each source, a direction field $u(x)$ is required [S1].
*   Grid resolution and domain bounds [S1].

**Outputs:**
*   A scalar light volume (energy density field) $\rho(x)$ [S1].

### Data Structures
The method operates on a regular 3D grid.
*   **Grid:** A 3D array storing the energy density $\rho$.
*   **Fields:**
    *   $\rho(x)$: Energy density at grid point $x$.
    *   $\sigma_a(x)$: Absorption coefficient at grid point $x$.
    *   $\sigma_s$: Scattering coefficient (scalar, assumed constant for isotropic scattering) [S1].
    *   $u(x)$: Light propagation direction field at point $x$. This is a vector field. For simple light sources (distant, point, cone), $u(x)$ is unique for direct lighting [S1].

### Governing Equation
The light propagation is modeled by the convection-diffusion equation [S1]:

$$
\frac{\partial}{\partial t} \rho (x) = -c u(x) \cdot \nabla \rho (x) - \sigma_a (x)\rho (x) + \nabla \cdot (D\nabla \rho (x))
$$

Where:
*   $\rho(x)$ is the energy density [S1].
*   $c$ is the speed of light [S1].
*   $u(x)$ is the direction field of light propagation [S1].
*   $\sigma_a(x)$ is the absorption coefficient [S1].
*   $D$ is the diffusion tensor [S1].

For isotropic scattering, the diffusion tensor is simplified to $D = \sigma_s I$, where $\sigma_s$ is the scattering coefficient and $I$ is the identity matrix. The equation becomes [S1]:

$$
\frac{\partial}{\partial t} \rho (x) = -c u(x) \cdot \nabla \rho (x) - \sigma_a (x)\rho (x) + \sigma_s \nabla^2 \rho (x)
$$

### Algorithm Steps
1.  **Initialization:** Initialize the energy density grid $\rho(x)$ to zero [G].
2.  **Direct Illumination (Convection):**
    *   For each of the $N$ light sources:
        *   Define the direction field $u(x)$ for the current light source. For a distant light, $u(x)$ is constant. For a point light at position $p$, $u(x) = \frac{x-p}{\|x-p\|}$ [S1].
        *   Solve the convection-diffusion equation (specifically the convection and absorption terms, as diffusion is handled separately for multiple scattering) to obtain the steady-state energy distribution $\rho_{direct}(x)$ for this light source. The convection term $-c u(x) \cdot \nabla \rho (x)$ models propagation, and $-\sigma_a (x)\rho (x)$ models absorption [S1].
        *   Accumulate $\rho_{direct}(x)$ into the global light volume $\rho_{total}(x)$ [S1].
3.  **Multiple Scattering (Diffusion):**
    *   Solve the diffusion equation using the accumulated light volume as the source or initial condition. The diffusion term $\sigma_s \nabla^2 \rho (x)$ models the spreading of light due to scattering [S1].
    *   This step produces the final light volume $\rho(x)$ that includes multiple scattering effects [S1].
4.  **Rendering:**
    *   Perform standard volume ray casting.
    *   For each ray, sample the light volume $\rho(x)$ at intersection points to determine the illumination contribution [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def grid_based_convection_diffusion(volume_data, light_sources, grid_resolution):
    """
    Computes the light volume using grid-based convection-diffusion.
    
    Args:
        volume_data: Object containing absorption coefficient sigma_a(x) for each voxel.
        light_sources: List of light source definitions. Each source has a type (distant, point) and parameters.
        grid_resolution: Tuple (nx, ny, nz) defining the grid dimensions.
        
    Returns:
        rho: 3D array (nx, ny, nz) representing the final light volume (energy density).
    """
    nx, ny, nz = grid_resolution
    
    # Initialize energy density grid to zero
    rho = zeros((nx, ny, nz))
    
    # Step 1: Direct Illumination (Convection) for each light source
    for light in light_sources:
        # Initialize temporary grid for this light source
        rho_direct = zeros((nx, ny, nz))
        
        # Define direction field u(x) based on light type
        # Note: This assumes simple light sources where u(x) is unique for direct lighting [S1]
        u_field = compute_direction_field(light, grid_resolution)
        
        # Solve convection equation to get steady state energy distribution
        # The equation is: d/dt rho = -c u . grad rho - sigma_a rho
        # We solve for the steady state (d/dt rho = 0) or iterate until convergence
        # rho_direct = solve_convection(rho_direct, u_field, volume_data.sigma_a, c)
        
        # Implementation detail: The source describes solving the equation numerically.
        # A simple explicit Euler scheme for steady state might look like:
        # rho_new = rho_old - dt * (-c * u . grad rho_old - sigma_a * rho_old)
        # However, the source does not specify the exact numerical solver (e.g., upwind, implicit).
        # We assume a generic solver function exists that returns the steady state.
        rho_direct = solve_steady_state_convection(
            initial_guess=zeros((nx, ny, nz)),
            direction_field=u_field,
            absorption_coefficient=volume_data.sigma_a,
            speed_of_light=c
        )
        
        # Accumulate into the global light volume
        rho = rho + rho_direct
        
    # Step 2: Multiple Scattering (Diffusion)
    # Solve diffusion equation: d/dt rho = sigma_s * laplacian(rho)
    # This step models multiple scattering [S1]
    # The source implies solving this on the accumulated light volume.
    # We assume an iterative solver or a single step if steady state is reached quickly.
    rho = solve_diffusion(
        initial_energy=rho,
        scattering_coefficient=volume_data.sigma_s,
        grid_resolution=grid_resolution
    )
    
    return rho

def compute_direction_field(light, grid_resolution):
    """
    Computes the direction field u(x) for a given light source.
    """
    nx, ny, nz = grid_resolution
    u_field = zeros((nx, ny, nz, 3))
    
    # Generate grid coordinates
    x, y, z = meshgrid(range(nx), range(ny), range(nz))
    coords = stack([x, y, z], axis=-1)
    
    if light.type == 'distant':
        # Constant direction
        u_field = light.direction
    elif light.type == 'point':
        # Direction from point light to voxel
        p = light.position
        vectors = coords - p
        norms = linalg.norm(vectors, axis=-1, keepdims=True)
        # Avoid division by zero
        norms = where(norms == 0, 1, norms)
        u_field = vectors / norms
    elif light.type == 'cone':
        # Direction from cone apex to voxel, clamped to cone angle
        # Implementation details for cone are not specified in sources [S1]
        # Placeholder for cone logic
        pass
        
    return u_field

def solve_steady_state_convection(initial_guess, direction_field, absorption_coefficient, speed_of_light):
    """
    Solves the convection-diffusion equation for direct lighting.
    The source does not specify the exact numerical scheme.
    This is a placeholder for the numerical solver [G].
    """
    # The source states: "solve a convection equation ... to obtain a steady state energy distribution" [S1]
    # A typical approach would be an upwind scheme for the convection term and implicit treatment for stability.
    # Since the source does not provide the specific algorithm, we return the result of a hypothetical solver.
    # In practice, this might involve solving a linear system (A * rho = b).
    return initial_guess # Placeholder

def solve_diffusion(initial_energy, scattering_coefficient, grid_resolution):
    """
    Solves the diffusion equation for multiple scattering.
    The source states: "solve a diffusion equation for multiple scattering within the volume" [S1]
    """
    # The source does not specify the exact numerical scheme for diffusion.
    # This is a placeholder for the numerical solver [G].
    return initial_energy # Placeholder
```

### Parameters
*   $N$: The number of light sources. The method performs $N$ convection solves [S1].
*   $\sigma_s$: The scattering coefficient. Used as the scalar value for the isotropic diffusion tensor $D = \sigma_s I$ [S1].
*   $\sigma_a(x)$: The spatially varying absorption coefficient [S1].
*   $u(x)$: The light propagation direction field. Assumed to be unique for direct lighting without scattering or reflection/refraction [S1].
*   $c$: The speed of light [S1].

### Pitfalls and Edge Cases
*   **Light Source Complexity:** The method assumes a unique light propagation direction for direct lighting. This holds for simple light sources (distant, point, cone) but fails for complex environment lights, which may have multiple incoming directions at a single point [S1].
*   **Performance:** The method is optimized for common light sources (distant/directional, point) and may not handle arbitrarily complex environment lights efficiently [S1].
*   **Numerical Stability:** The convection term $-c u(x) \cdot \nabla \rho (x)$ can be numerically unstable if not handled with appropriate schemes (e.g., upwinding), though the source does not specify the exact numerical method [S1].

## Operating characteristics
*   **Time Complexity:** The method requires solving the convection equation $N$ times (once per light source) and then solving the diffusion equation once [S1]. The complexity depends on the grid resolution and the numerical solver used.
*   **Memory Complexity:** Requires storing the energy density grid $\rho(x)$, the absorption coefficient grid $\sigma_a(x)$, and the direction field $u(x)$ for each light source if computed on the fly or stored [S1].
*   **Performance:** The method achieves real-time performance by using highly parallelized grid-based numerical schemes, unlike Monte Carlo ray tracing which is prohibitively expensive [S1].
*   **Strengths:** Produces realistic multiple scattering effects, which SAT techniques cannot do [S1]. Supports dynamic transfer function editing and various light sources [S1].
*   **Limitations:** Limited to simple light sources for the convection step. Not suitable for complex environment lighting without modification [S1].

## Background
The method was proposed in the paper "Fast global illumination for interactive volume visualization" [S1]. It builds on the idea of separating illumination modeling and simulation, a concept also seen in Precomputed Radiance Transfer (PRT) methods, but adapts it for volume visualization where local and internal light sources are more important than complex distant lighting [S1]. The technique is designed to meet the goals of real-time rendering, support for dynamic transfer functions, and handling of multiple common light sources [S1]. It addresses the limitations of earlier methods like Kajiya-Von Herzen ray tracing (high computational cost) and SAT techniques (limited multiple scattering) [S1].

## Sources
[S1] Fast global illumination for interactive volume visualization, https://doi.org/10.1145/2448196.2448205, fetched 2026-10-30
