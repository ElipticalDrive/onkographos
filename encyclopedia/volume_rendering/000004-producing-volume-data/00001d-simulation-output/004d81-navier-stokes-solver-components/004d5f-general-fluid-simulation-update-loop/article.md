# General Fluid Simulation Update Loop

Also called: Fluid Simulation Update Loop

## Overview
The General Fluid Simulation Update Loop is the core iterative process that advances a fluid simulation state from time $t$ to $t + dt$. It solves the incompressible Navier-Stokes equations by sequentially applying source injection, diffusion, pressure projection, and advection to velocity and density fields [S1][S2, unofficial]. Unlike specialized variants like Stable Fluid Dynamics for Computer Graphics which approximate shallow water equations, this loop handles general 3D incompressible flow on a grid [S2, unofficial]. It serves as the bridge between physical simulation and rendering, often feeding data into particle systems or volume renderers [S1].

## How it works
The loop operates on a staggered or collocated grid storing velocity $\mathbf{u}$ and scalar fields (density $\rho$, temperature, etc.). The standard sequence, as observed in general implementations, is:

1.  **Source Injection**: Add external forces to velocity and sources to density.
2.  **Diffusion**: Solve the viscous diffusion equation.
3.  **Projection**: Enforce incompressibility ($\nabla \cdot \mathbf{u} = 0$) by solving a Poisson equation for pressure.
4.  **Advection**: Transport velocity and density fields by the velocity field.
5.  **Particle Step (Optional)**: If hybrid, step particles using grid velocity, sort them, and accumulate their contribution back to the grid [S1].
6.  **Render**: Output the density or surface field [S1].

### Data Structures
*   **Grid**: A 3D array of size $N_x \times N_y \times N_z$.
*   **Velocity Field ($\mathbf{u}$)**: Can be collocated (scalar at each cell) or staggered (components on faces). The parent article [Navier-Stokes Solver Components](13501-navier-stokes-solver-components.md) details the specific layout.
*   **Density Field ($\rho$)**: Scalar field per cell.
*   **Pressure Field ($p$)**: Scalar field per cell.
*   **Particle Buffer**: List of particles with position $\mathbf{x}$, velocity $\mathbf{v}$, and mass/density contribution.

### Pseudocode
The following pseudocode assumes a semi-Lagrangian advection scheme and a standard projection step. It integrates the steps described in [S1] and [S2, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fluid_update_loop(grid, particles, dt, viscosity, diffusion_rate):
    # 1. Inject Sources [S1]
    # Apply external forces (gravity, user input) to velocity
    grid.velocity += dt * grid.external_forces
    # Add density sources (smoke, dye)
    grid.density += dt * grid.density_sources

    # 2. Diffusion [S2, unofficial]
    # Solve implicit diffusion equation: u_new - dt * nu * laplacian(u_new) = u_old
    # This typically involves solving a linear system (e.g., Gauss-Seidel or conjugate gradient)
    grid.velocity = solve_diffusion(grid.velocity, viscosity, dt)
    grid.density = solve_diffusion(grid.density, diffusion_rate, dt)

    # 3. Projection (Pressure Solve) [S1][S2, unofficial]
    # Enforce divergence-free constraint: div(u) = 0
    # Solve Poisson equation: laplacian(p) = div(u) / dt
    pressure = solve_poisson(grid.velocity, dt)
    # Subtract pressure gradient from velocity
    grid.velocity -= dt * gradient(pressure)
    
    # Optional: Correct velocity field using divergence and pressure iteration [S1]
    # This is often part of the projection solver itself.

    # 4. Advection [S1][S2, unofficial]
    # Advect velocity by itself: du/dt = -(u . grad)u
    # Advect density by velocity: drho/dt = -(u . grad)rho
    # Using semi-Lagrangian method: trace back from current cell center
    grid.velocity = advect_semi_lagrangian(grid.velocity, grid.velocity, dt)
    grid.density = advect_semi_lagrangian(grid.density, grid.velocity, dt)

    # 5. Particle Step (if hybrid) [S1]
    if particles:
        # Step particles from grid velocity
        for p in particles:
            p.velocity = interpolate_grid_velocity(grid.velocity, p.position)
            p.position += dt * p.velocity
        
        # Sort particles by grid cell for efficient accumulation
        sort_particles_by_cell(particles)
        
        # Accumulate particle density/velocity to grid
        clear_grid_accumulators(grid)
        for p in particles:
            grid_accumulate(grid, p.position, p.velocity, p.mass)
        
        # Add accumulated values to main fields
        grid.velocity += grid.velocity_accumulator
        grid.density += grid.density_accumulator

    # 6. Render [S1]
    # Render density or surface field
    render_volume(grid.density)
```

### Parameters
*   **$dt$**: Time step. Must satisfy CFL condition for explicit schemes, but implicit diffusion allows larger steps. Typical values: $0.01$ to $0.1$ seconds for real-time, smaller for high-accuracy offline.
*   **$\nu$ (Viscosity)**: Kinematic viscosity. Typical values: $0.00001$ for water, $0.0001$ for oil.
*   **Diffusion Rate**: For density. Often set equal to viscosity or higher for visual diffusion.

### Edge Cases and Pitfalls
*   **Precision**: Floating point errors in projection can lead to divergence drift. Re-projection or higher precision solvers may be needed [S1].
*   **Boundaries**: No-slip (velocity=0) or free-slip (normal velocity=0, tangential derivative=0) conditions must be enforced at grid boundaries.
*   **Degenerate Input**: Zero velocity fields can cause division by zero in advection if not handled.
*   **Concurrency**: In GPU implementations, simultaneous writes to neighboring voxels require atomic operations or double-buffering [S3, unofficial].

## Subtopics

### Stable Fluids (2D)
A simplified 2D variant that updates fields sequentially: diffusion, projection, advection. It corrects velocity using divergence and pressure iteration during projection. It is chosen for real-time 2D interactive applications where performance is critical and 3D complexity is unnecessary [S1].

### Stable Fluid Dynamics for Computer Graphics
A method for animating water based on a stable solution of partial differential equations approximating the shallow water equations. It results in a wave equation on a height-field where wave velocity is proportional to the square root of water depth. It is solved using an alternating-direction implicit method on a uniform finite-difference grid. Each iteration consists mainly of solving a simple tridiagonal linear system for each row and column of the height field. A single iteration per frame suffices in most cases for convincing animation. It renders water using caustic shading to simulate refraction and uses a wetness map for wetting/drying of sand. The height-field representation prevents easy simulation of breaking waves, except perhaps in combination with particle-based fluid models [S2, unofficial].

### Cellular Automata Simulation
A discrete, grid-based approach using simple rules rather than continuous PDEs. Water gravity checks if the voxel underneath is air; if so, it moves there. Sideways movement uses a hash of position and frame number for pseudo-random direction, moving only if the target is not solid. Direction persistence stores the first successful horizontal direction until a wall is hit, with a small probability of changing direction for natural appearance. Lava logic runs the same movement code but converts lava to stone if water is found in its von Neumann neighborhood (up, down, left, right). Sand logic only falls down directly or diagonally if obstructed, pushing up liquids it falls onto. This method is used for voxel-based simulations where simplicity and parallelism are prioritized over physical accuracy [S3, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N)$ per step for advection and diffusion (where $N$ is the number of grid cells), assuming linear solvers. Projection step depends on the solver (e.g., $O(N)$ for multigrid, $O(N \log N)$ for FFT).
*   **Memory Complexity**: $O(N)$ for grid fields. Particle systems add $O(P)$ where $P$ is the number of particles.
*   **Measured Performance**: Real-time 2D simulations can run at 60+ FPS on CPU. 3D simulations typically require GPU acceleration (CUDA/OpenCL) for real-time performance at reasonable resolutions (e.g., $128^3$).
*   **Scaling**: Scales well with grid resolution on GPUs due to parallel nature of advection and diffusion.
*   **Strengths**: Physically plausible, handles complex interactions, versatile for various fluid types.
*   **Limitations**: Computationally expensive for high-resolution 3D, requires careful tuning of parameters, stability issues with large time steps in explicit schemes.
*   **When to Use**: General-purpose fluid simulation, smoke, fire, water.
*   **When Not to Use**: When breaking waves are critical (use height-field or particle methods), or when extreme performance is needed with low accuracy (use cellular automata).

## Background
The general fluid simulation update loop is rooted in the work of Jos Stam on "Stable Fluids" [S2, unofficial], which introduced stable, real-time fluid simulation for computer graphics. The specific sequence of injection, diffusion, projection, and advection is a standard discretization of the Navier-Stokes equations. The hybrid particle-grid approach is seen in modern implementations like those by Razria93 [S1]. Cellular automata approaches are popular in voxel-based games and simulations [S3, unofficial].

## Sources
[S1] Part4 Chapter16 Fluid Simulation Demo, https://github.com/Razria93/Graphics_Study_Archive/issues/33, fetched 2026-10-26
[S2] PDF Volume Rendering Methods for Computational Fluid Dynamics Visualization, https://www.researchgate.net/profile/David-Ebert-4/publication/220943451_Volume_Rendering_Methods_for_Computational_Fluid_Dynamics_Visualization/links/00b7d51a73e2892e23000000/Volume-Rendering-Methods-for-Computational-Fluid-Dynamics-Visualization.pdf, fetched 2026-10-27 (unofficial)
[S3] Ray-Marched Voxel Simulation - Joris Rijsdijk, https://www.jorisar.nl/blog/VoxelPlayground/, fetched 2026-10-25 (unofficial)
