# Particle Motion with Thermal Buoyancy and Diffusion

Also called: Thermal Buoyancy and Diffusion Model

## Overview
This method defines the kinematic trajectory of massless particles in volumetric fire simulation by combining wind advection, thermal buoyancy, source movement, and diffusion. It solves the problem of realistic macro-scale flame motion by treating particles as tracer elements subject to specific physical forces, building on the [Massless Particle System](3970-massless-particle-system.md) framework. It is distinguished from pure fluid solvers by its reliance on simplified Navier-Stokes approximations for wind and explicit algebraic formulas for buoyancy, prioritizing real-time performance over strict physical fidelity.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Particle position $x_p$, particle temperature $T_p$, particle age $t_p$, ambient temperature $T_o$, wind field $w(x_p, t)$, source velocity $V_p$, thermal coefficient $\beta$ (or $b$), gravitational acceleration $g$, and kinematic viscosity $\nu$.
*   **Outputs:** Updated particle position $x_p(t + \Delta t)$.

**Data Structures:**
The particle state is typically stored in a structure containing:
*   `position`: 3D vector $(x, y, z)$.
*   `temperature`: Scalar float $T_p$.
*   `age`: Scalar float $t_p$.
*   `creation_time`: Scalar float (used to compute $t_p$).

**Formulas:**
The primary equation of motion for a particle is [S1, unofficial][S2]:

$$ \frac{dx_p}{dt} = w(x_p, t) + d(T_p) + V_p + c(T_p, t) $$

Where:
*   $w(x_p, t)$ is the arbitrary controlling wind field.
*   $d(T_p)$ is the motion due to diffusion, modeled as temperature-scaled Brownian motion.
*   $V_p$ is the motion due to the movement of the fire source.
*   $c(T_p, t)$ is the motion due to thermal buoyancy.

**Thermal Buoyancy:**
Thermal buoyancy is constant over the lifetime of the particle [S1, unofficial][S2]:

$$ c(T_p, t) = -\beta g (T_o - T_p) t_p^2 $$

Note: Source [S1, unofficial] uses the coefficient $b$ and includes a factor $y$ in the formula $c(T_p, t) = -b g y (T_o - T_p) t_p^2$, while [S2] uses $\beta$ and omits $y$. The term $(T_o - T_p)$ indicates that if $T_p > T_o$ (hot particle), the force is negative (upward, assuming $g$ is positive magnitude and $y$ is up, or vice versa depending on coordinate system convention). The sources disagree on the exact scalar coefficients and signs; implementers should verify the coordinate system orientation. Typically, $\beta$ is the thermal expansion coefficient.

**Wind Field:**
The wind field $w$ is computed using a reduced form of the Navier-Stokes equations [S2]:

$$ w(u) = \nu \nabla \cdot (\nabla u) - (u \cdot \nabla)u - \nabla p $$

Where:
*   $u$ is a 4D vector $(x_p, t)$ representing spatial dimensions and time [S2].
*   $\nu$ is the kinematic viscosity.
*   $p$ is pressure.
*   The term $(u \cdot \nabla)u$ represents convection.
*   The term $\nu \nabla \cdot (\nabla u)$ represents drag/diffusion.

**Parameters:**
*   $\beta$ (or $b$): Thermal coefficient scaling buoyancy force [S1, unofficial][S2].
*   $g$: Gravitational acceleration constant [S1, unofficial][S2].
*   $T_o$: Ambient temperature of the surrounding air [S1, unofficial][S2].
*   $t_p$: Age of the particle, calculated as current time minus creation time [S1, unofficial][S2].
*   $\nu$: Kinematic viscosity, controlling the rotation and smoothness of the wind field [S2].

**Pitfalls:**
*   The model assumes molecular diffusion is negligible relative to other effects [S2].
*   The gas is assumed to be incompressible [S2].
*   Applying full Navier-Stokes equations would be computationally prohibitive for real-time rendering, hence the reduced form [S2].
*   Precision issues may arise with $t_p^2$ for very old particles if not clamped or handled.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_particle(particle, dt, wind_field, source_velocity, ambient_temp, gravity, thermal_coeff, viscosity):
    """
    Updates a single particle's position based on thermal buoyancy and diffusion model.
    
    Args:
        particle: Object with attributes .pos (vec3), .temp (float), .age (float).
        dt: Time step.
        wind_field: Function w(pos, time) returning vec3.
        source_velocity: vec3 V_p.
        ambient_temp: float T_o.
        gravity: float g (magnitude).
        thermal_coeff: float beta (or b).
        viscosity: float nu.
        
    Returns:
        Updated particle object.
    """
    # 1. Compute Thermal Buoyancy
    # c(Tp, t) = -beta * g * (To - Tp) * tp^2
    # Note: Sign convention depends on coordinate system. 
    # Assuming standard physics: hot air rises. If g is vector (0, -9.8, 0), 
    # then (To - Tp) is negative for hot particles, making c positive (up).
    # Source [S1, unofficial] has extra 'y' factor; Source [S2] does not. Using [S2] form as primary.
    buoyancy_force = -thermal_coeff * gravity * (ambient_temp - particle.temp) * (particle.age ** 2)
    
    # 2. Compute Wind Field
    # w(u) = nu * Laplacian(u) - (u . grad)u - grad(p)
    # This requires a spatial solver or approximation. 
    # For implementation, wind_field(pos, time) is assumed to return the result of this computation.
    wind_velocity = wind_field(particle.pos, particle.age)
    
    # 3. Compute Diffusion
    # d(Tp) is temperature-scaled Brownian motion.
    # Not explicitly defined in sources beyond name. 
    # Common implementation: random vector scaled by temperature.
    # [G] Assuming standard Brownian motion approximation:
    diffusion_motion = random_gaussian_vector() * particle.temp
    
    # 4. Source Movement
    source_motion = source_velocity
    
    # 5. Total Velocity
    total_velocity = wind_velocity + diffusion_motion + source_motion + buoyancy_force
    
    # 6. Update Position
    particle.pos += total_velocity * dt
    
    # 7. Update Age
    particle.age += dt
    
    return particle
```

## Subtopics

### Physics-Based Macro Flame Animation
This variant focuses on the macro-movement of flame spines using the system of equations from [Kan99] as cited in [S2]. It emphasizes the integration of the thermal buoyancy and diffusion model within a game engine context, leveraging shader software for volumetric rendering. It distinguishes itself by explicitly linking the particle motion to "flame spines" and bounding volumes composed of hexahedrons, rather than just individual tracer particles. It is chosen when the visual requirement is large-scale, coherent flame structures rather than fine-grained turbulent details.

## Operating characteristics
*   **Time Complexity:** $O(N)$ for $N$ particles, assuming the wind field $w(x_p, t)$ can be evaluated in $O(1)$ or precomputed. The buoyancy calculation is $O(1)$.
*   **Memory Complexity:** $O(N)$ to store particle states (position, temperature, age).
*   **Performance:** Designed for real-time rendering. The use of reduced Navier-Stokes equations avoids the computational cost of full fluid simulation [S2].
*   **Strengths:** Computationally efficient; provides physically plausible macro-motion for fire; easy to integrate with existing particle systems.
*   **Limitations:** Lacks fine-scale turbulence details (handled by diffusion/noise); assumes incompressible flow; accuracy depends on the quality of the wind field approximation.
*   **When to use:** Real-time volumetric fire rendering in games or interactive applications.
*   **When not to use:** Scientific simulations requiring high-fidelity fluid dynamics or compressible flow effects.

## Background
The method originates from research on realistic real-time rendering of fire in production systems. It was presented by Y. Vanzine and D. Vrajitoru in 2008 [S2] and in an earlier proposal [S1, unofficial]. The model draws inspiration from [FKM* 07] for volumetric rendering integration and uses physics-based controls from [Kan99] for macro-animation. It is used in game engines and shader-based volumetric rendering pipelines.

## Sources
[S1] Realistic Real-Time Rendering of Fire in a Production System, https://clas.iusb.edu/math-compsci/_prior-proposals/YVanzine_Proposal.pdf, fetched 2026-10-28 (unofficial)
[S2] Pseudorandom Noise for Real-Time Volumetric Rendering of Fire in a Production System, https://doi.org/10.2312/vg/vg-pbg08/129-136, fetched 2026-10-26
