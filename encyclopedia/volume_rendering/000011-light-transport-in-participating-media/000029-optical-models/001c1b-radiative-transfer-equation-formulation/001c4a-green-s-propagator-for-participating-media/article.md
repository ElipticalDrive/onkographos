# Green's Propagator for Participating Media

Also called: Propagator G, Radiative Transfer Propagator

## Overview
The Green's propagator $G$ is the fundamental solution to the homogeneous radiative transfer equation, representing the radiance at a point $\mathbf{x}$ in direction $\vec{\omega}$ resulting from a point source at $\mathbf{x}'$ emitting in direction $\vec{\omega}'$ [S1]. It serves as the kernel for light transport in participating media, generalizing the concept of ray tracing with attenuation to include scattering [S1]. Unlike the [Standard volume rendering equation](1165-standard-volume-rendering-equation.md) which integrates emission and scattering directly, the propagator formulation separates the transport physics (attenuation and propagation) from the source terms, enabling iterative solutions like the Neumann series [S2, unofficial].

## How it works
**Inputs:**
*   $\mathbf{x}$: Current position in the medium.
*   $\vec{\omega}$: Current direction of propagation.
*   $\mathbf{x}'$: Source position.
*   $\vec{\omega}'$: Source direction.
*   $s$: Path length traveled from the source.
*   $c(\mathbf{x})$: Extinction coefficient (absorption + scattering) at position $\mathbf{x}$ [S1].
*   $b(\mathbf{x}, \vec{\omega}, \vec{\omega}')$: Scattering phase function and coefficient [S1].

**Outputs:**
*   $G(s, \mathbf{x}, \vec{\omega}, \mathbf{x}', \vec{\omega}')$: Radiance at $\mathbf{x}$ in direction $\vec{\omega}$ due to the point source [S1].

**Data Structures:**
The propagator is evaluated along ray segments. For discrete implementation, it requires sampling the extinction coefficient along the path [S1].

**Steps:**
1.  **Initialize Initial Condition:** At path length $s=0$, the propagator is a Dirac delta function representing the point source [S1]:
    $$ G(0, \mathbf{x}, \vec{\omega}, \mathbf{x}', \vec{\omega}') = \delta(\mathbf{x} - \mathbf{x}')\delta(\vec{\omega} - \vec{\omega}') $$
2.  **Evaluate No-Scatter Propagator:** In the absence of scattering ($b=0$), the propagator describes light traveling in a straight line with exponential attenuation [S1]:
    $$ G_{\text{noscatter}}(\mathbf{x}, \vec{\omega}, \mathbf{x}', \vec{\omega}') = \delta(\mathbf{x} - \vec{\omega}s - \mathbf{x}')\delta(\vec{\omega} - \vec{\omega}') \exp\left(-\int_0^s c(\mathbf{x} - \vec{\omega}(s - s')) ds'\right) $$
    The term $\delta(\mathbf{x} - \vec{\omega}s - \mathbf{x}')$ ensures the evaluation point lies on the ray originating from $\mathbf{x}'$ in direction $\vec{\omega}'$ [S1]. The integral computes the optical depth along the path from the source to $\mathbf{x}$ [S1].
3.  **Include Scattering (General Case):** For media with scattering, $G$ is the solution to the homogeneous RTE [S1]. It can be expanded via the Neumann series, where the zeroth-order term is $G_{\text{noscatter}}$ and higher-order terms account for single, double, etc., scattering events [S2, unofficial]. The no-scatter case admits a closed-form expression, while the general scattering case requires iterative integration or approximation [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def green_propagator_no_scatter_dda(origin, direction, max_dist, extinction_fn, grid_bounds_min, grid_bounds_max):
    """
    Computes the Green's propagator attenuation along a ray using a robust 3D DDA traversal.
    Handles negative directions, zero direction components, and grid boundaries correctly.
    """
    # Initialize DDA state
    t = 0.0
    optical_depth = 0.0
    
    # Precompute inverse direction and initial step
    inv_dir = [1.0 / d if abs(d) > 1e-8 else 1e8 for d in direction]
    step = [1 if d > 0 else -1 for d in direction]
    
    # Initial cell coordinates
    current_cell = [int((origin[i] - grid_bounds_min[i]) * inv_dir[i]) if abs(direction[i]) > 1e-8 else int((origin[i] - grid_bounds_min[i]) / (grid_bounds_max[i] - grid_bounds_min[i]) * 1000) for i in range(3)]
    
    # Distance to next grid boundary in each axis
    t_max = [0.0, 0.0, 0.0]
    for i in range(3):
        if abs(direction[i]) > 1e-8:
            boundary = grid_bounds_min[i] + (current_cell[i] + (1 if step[i] > 0 else 0)) * (grid_bounds_max[i] - grid_bounds_min[i]) / 1000.0
            t_max[i] = (boundary - origin[i]) * inv_dir[i]
        else:
            t_max[i] = float('inf')
            
    # DDA traversal loop
    while t < max_dist:
        # Find the axis with the smallest t_max
        axis = 0
        if t_max[1] < t_max[axis]: axis = 1
        if t_max[2] < t_max[axis]: axis = 2
        
        # Advance to next boundary
        dt = t_max[axis] - t
        t = t_max[axis]
        
        # Sample extinction at current position and accumulate optical depth
        pos = [origin[i] + direction[i] * t for i in range(3)]
        optical_depth += extinction_fn(pos) * dt
        
        # Update t_max for the traversed axis
        t_max[axis] += (grid_bounds_max[axis] - grid_bounds_min[axis]) / 1000.0 * inv_dir[axis]
        
        # Move to next cell
        current_cell[axis] += step[axis]
        
        # Check bounds
        if any(current_cell[i] < 0 or current_cell[i] > 1000 for i in range(3)):
            break
            
    return exp(-optical_depth)
```

**Parameters:**
*   $c(\mathbf{x})$: Extinction coefficient. Typical values depend on the medium (e.g., fog, smoke). Not given in the sources.
*   $s$: Path length. Determined by the geometry.

**Edge Cases:**
*   **Precision:** The Dirac delta functions require careful handling in discrete implementations. In practice, the propagator is evaluated along ray segments where the geometric constraints are satisfied [S1].
*   **Boundaries:** The no-scatter formula assumes straight-line propagation, which fails if the medium has boundaries that cause refraction or reflection (not covered in the no-scatter model) [S1].

## Subtopics

### Transport Operator
The Transport Operator $\mathbf{G}$ is an operator that acts on an outgoing radiance field $g$ to produce the incoming radiance field at a point, accounting for attenuation along the ray [S2, unofficial]. It integrates the radiance from the surface or medium boundary to the current point, including attenuation [S2, unofficial].
**Formula:**
$$ <\mathbf{G} g>(\mathbf{x}, \vec{\omega}) := \int_0^{\|\mathbf{x}_S - \mathbf{x}\|} T(\mathbf{x}', \mathbf{x}) g(\mathbf{x}', -\vec{\omega}) \, ds + T(\mathbf{x}_S, \mathbf{x}) g(\mathbf{x}_S, -\vec{\omega}) $$
where $T(\mathbf{x}', \mathbf{x})$ is the transmittance from $\mathbf{x}'$ to $\mathbf{x}$, and $\mathbf{x}_S$ is the surface intersection point [S2, unofficial]. This operator is used in the operator notation of the Volume Rendering Equation: $L_i = \mathbf{G}(L_e + \mathbf{K} L_i)$ [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **No-scatter:** $O(N)$ where $N$ is the number of samples along the ray to compute the optical depth integral. Memory is $O(1)$ per ray [S1].
*   **With scattering:** Complexity depends on the number of scattering events considered. The Neumann series expansion leads to $O(N^k)$ for $k$ scattering events if computed naively, but can be optimized with techniques like photon mapping or path tracing [S2, unofficial]. Not given in the sources for specific complexities.

**Measured Performance:**
*   The no-scatter propagator is equivalent to simple raytracing with fog attenuation, a model popular in computer graphics and part of APIs like OpenGL [S1]. This is very fast and widely supported.

**Strengths and Limitations:**
*   **Strengths:** Provides a clear physical interpretation of light transport. The no-scatter case is computationally efficient and easy to implement [S1]. The operator formulation allows for a compact mathematical representation of the RTE [S2, unofficial].
*   **Limitations:** The general scattering case is complex and requires iterative solutions or approximations. The no-scatter model ignores scattering effects, which are significant in many participating media [S1].

**When to use:**
*   Use the no-scatter propagator for simple fog, haze, or volumetric effects where scattering is negligible or approximated as absorption [S1].
*   Use the general propagator or its operator formulation for accurate rendering of multiple scattering effects in media like smoke, clouds, or biological tissue [S1][S2, unofficial].

## Background
The Green's propagator formulation originates from the radiative transfer equation, a fundamental equation in physics describing light transport in participating media [S1]. The specific formulation and its relation to raytracing are discussed in "Practical Rendering of Multiple Scattering Effects in Participating Media" by Raviraj et al. [S1]. The operator notation, including the Transport Operator, is detailed in "Volume Rendering Equation (VLTE) 2" [S2, unofficial].

## Sources
[S1] Practical Rendering of Multiple Scattering Effects in Participating Media, https://cseweb.ucsd.edu/~ravir/HRPIEG.pdf, fetched 2026-10-28
[S2] ボリュームレンダリング方程式(Volume Rendering Equation) 2, https://rayspace.xyz/notes/VLTE2/, fetched 2026-10-30 (unofficial)
