# Massless Particle System

Also called: Drag-Dominated Particle System, Massless Particle Tracing, Particle-Based Smoke Visualization

## Overview
The Massless Particle System visualizes flow fields by tracking tracer particles that move through a simulation domain influenced by an underlying velocity field [S1, unofficial]. It solves the problem of representing continuous fluid motion with discrete elements, operating under the assumption that the drag force on the particle is large compared to gravity [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A velocity field $V(x, t)$ computed by a solver (e.g., FDS) [S1, unofficial].
*   **Output:** Particle positions $x_p(t)$ and optionally streak lines representing trajectory history [S1, unofficial].

**Data Structures:**
*   **Particle State:** A position vector $x_p$, a timestamp, and a history buffer storing past positions [S1, unofficial].
*   **Velocity Field:** A discrete grid or analytic function providing $V(x, t)$ [S1, unofficial].

**Parameters:**
*   **Particle Size:** Specified as a floating-point value `partpointsize` [S1, unofficial].
*   **Streak Length:** Specified as a time duration in seconds before the current display time [S1, unofficial].
*   **Note:** These parameters are configured via interface dialogs such as File/Bounds [S1, unofficial].

**Method Steps:**
1.  **Initialization:** Place particles at initial positions $x_p(0)$ within the simulation domain [S1, unofficial].
2.  **Integration:** Update particle position using the differential equation:
    $$ \frac{dx_p}{dt} = V(x_p, t) $$
    where $x_p$ is the particle position and $V$ is the velocity at that position and time [S1, unofficial].
3.  **Rendering:** Render particles as points with size `partpointsize` or as streak lines extending back in time by the specified streak length [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_massless_particles(particles, velocity_field, dt, streak_length_seconds):
    for p in particles:
        v = velocity_field(p.pos)
        p.pos = p.pos + v * dt
        p.history.append(p.pos)
        # Trim history to match streak_length_seconds
        max_steps = max(1, int(streak_length_seconds / dt))
        while len(p.history) > max_steps:
            p.history.pop(0)
    return particles

def render_particles(particles, partpointsize, streak_length_seconds):
    # Configure graphics pipeline point size
    graphics_state.point_size = partpointsize
    for p in particles:
        # Submit vertex for current position
        graphics_pipeline.submit_vertex(p.pos)
        graphics_pipeline.draw_points()
        # Submit line strip for history
        if streak_length_seconds > 0 and len(p.history) > 1:
            graphics_pipeline.submit_line_strip(p.history)
            graphics_pipeline.draw_lines()
```

**Edge Cases and Pitfalls:**
*   **Gravity Assumption:** The model assumes drag force is large compared to gravity ($mg$) [S1, unofficial]. If gravity is significant, this model is inaccurate [S1, unofficial].
*   **Static Images:** Static particle images do not effectively display motion; streak lines are preferred for visualizing curved motion due to obstructions [S1, unofficial].
*   **Precision:** The accuracy of individual particle motion is less critical than the statistical behavior of the ensemble, allowing for simpler integration methods [S2].

## Subtopics

### Hotaru
A visualization method where thousands of tracer particles are randomly scattered in the whole simulation data space to show statistical behavior [S2]. It is distinguished by its focus on statistical representation rather than precise trajectory accuracy, using the 1st order Euler method for numerical integration [S2]. Choose this when the goal is to visualize the general flow pattern or statistical behavior of a vector field rather than individual particle paths [S2].

### [Particle Motion with Thermal Buoyancy and Diffusion](14660-particle-motion-with-thermal-buoyancy-and-diffusion.md)
This method defines the kinematic trajectory of massless particles in volumetric fire simulation by combining wind advection, thermal buoyancy, source movement, and diffusion. It solves the problem of realistic macro-scale flame motion by treating particles as tracer elements subject to specific physical forces, building on the [Massless Particle System](3970-massless-particle-system.md) framework. It is distinguished from pure fluid solvers by its reliance on simplified Navier-Stokes approximations for wind and explicit algebraic formulas for buoyancy, prioritizing real-time performance over strict physical fidelity.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Computationally inexpensive; effective for visualizing flow patterns and obstructions via streak lines [S1, unofficial].
*   **Limitations:** Does not model particle inertia or gravity accurately; relies on an external velocity field [S1, unofficial].
*   **When to Use:** When visualizing smoke flow or wind patterns where drag dominates and real-time performance is needed [S1, unofficial].

## Background
The method was implemented in Smokeview (Version 6) for visualizing Fire Dynamics Simulation (FDS) data [S1, unofficial]. It was the first method implemented in Smokeview for visualizing smoke flow and was inspired by the software tool named Frames written by James Sims [S1, unofficial]. The approach uses particles as tracer elements to allow visualization of flow, distinct from droplets used for fire suppression modeling or fuel elements like embers [S1, unofficial].

## Sources
[S1] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
[S2] IMMERSIVE VR VISUALIZATIONS BY VFIVE PART 1: DEVELOPMENT, https://doi.org/10.1142/s1793962313400035, fetched 2026-10-29
