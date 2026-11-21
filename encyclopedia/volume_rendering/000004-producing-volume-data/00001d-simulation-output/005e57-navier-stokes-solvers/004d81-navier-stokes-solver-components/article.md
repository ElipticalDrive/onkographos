# Navier-Stokes Solver Components

## Overview
This family provides the numerical discretization and solution steps for the incompressible Navier-Stokes equations on a grid, producing the velocity and scalar fields required for fluid simulation [S1, unofficial][S2, unofficial]. It solves the problem of advancing fluid state over time by combining advection, diffusion, and pressure projection, distinguishing it from [Grid-Based Simulation](13502-grid-based-simulation.md) by its explicit component breakdown and from [Navier-Stokes Fire Simulation](14659-navier-stokes-fire-simulation.md) by its general-purpose incompressible flow focus.

## Shared principles
Builds on [Navier-Stokes Solvers](17619-navier-stokes-solvers.md), which defines the shared grid layout, velocity advection, pressure projection, diffusion, and time integration steps for this family [S1, unofficial][S2, unofficial].

## Subtopics

### [Smoke Source Injection](2494-smoke-source-injection.md)
Initializes a volumetric smoke source at a fixed grid location by setting density and velocity values within a defined spatial region. It differs from [Smoke Advection](4639-smoke-advection.md) by creating new data rather than moving existing data, and from the [General Fluid Simulation Update Loop](13472-general-fluid-simulation-update-loop.md) by being a boundary/initial condition step rather than a time-integration step. Choose it when initializing a simulation or resetting a region with new mass and momentum. See [Smoke Source Injection](2494-smoke-source-injection.md).

### [Smoke Advection](4639-smoke-advection.md)
Tracks scalar fields (density, temperature, color) transported by the fluid velocity field to visualize motion, as velocity alone lacks visual detail. It differs from velocity advection by solving the passive scalar transport equation $\frac{\partial \phi}{\partial t} = -(\mathbf{u} \cdot \nabla) \phi$ using the same numerical scheme as the velocity, allowing low-resolution grids to retain visual detail. Unlike the unconditionally stable semi-Lagrangian scheme, the MacCormack scheme used here is conditionally stable and requires a limiter to prevent numerical blow-up. Choose it when visualizing fluid motion via passive or buoyant scalars on low-resolution grids. See [Smoke Advection](4639-smoke-advection.md).

### [General Fluid Simulation Update Loop](13472-general-fluid-simulation-update-loop.md)
The core iterative process that advances a fluid simulation state from time $t$ to $t + dt$ by sequentially applying source injection, diffusion, pressure projection, and advection to velocity and density fields. Unlike specialized variants like Stable Fluid Dynamics for Computer Graphics which approximate shallow water equations, this loop handles general 3D incompressible flow on a grid. It serves as the bridge between physical simulation and rendering, often feeding data into particle systems or volume renderers. Choose it for general 3D incompressible flow requiring a complete time-integration step. See [General Fluid Simulation Update Loop](13472-general-fluid-simulation-update-loop.md).

### Buoyant Force for Smoke
- **Definition**: Adding a buoyant force to the velocity field based on temperature differences to make hot smoke rise and cool smoke fall [S1, unofficial].
- **Step**: Add the buoyant force to the velocity field immediately following velocity advection [S1, unofficial].
- **Formula**: $\mathbf{f}_{\text{buoyancy}} = \frac{Pmg}{R} \left( \frac{1}{T_0} - \frac{1}{T} \right) \mathbf{z}$, where $P$ is pressure, $m$ is molar mass, $g$ is gravity, $R$ is the universal gas constant, $T_0$ is ambient temperature, $T$ is advected temperature, and $\mathbf{z}$ is the normalized upward vector [S1, unofficial].
- **Parameter**: Physical constants $P, m, g, R$ can be treated as a single value and tweaked for visual appearance [S1, unofficial].

### Reduced Navier-Stokes Wind Field Simulation
- **Definition**: A physics-based framework for modeling gaseous flow (wind field) that incorporates only physical elements corresponding to interesting visual effects, not scientific accuracy [S2, unofficial]. It represents a volume of gas as a combination of a scalar temperature field, a scalar pressure field, and a vector velocity field [S2, unofficial].
- **Step**: Model rotational, buoyant, and convective components of gaseous motion by coupling a reduced form of the Navier-Stokes equations with an equation for turbulent mixing due to temperature differences in a gas [S2, unofficial].
- **Formula**: $\frac{\partial \mathbf{u}}{\partial t} = \nu \nabla^2 \mathbf{u} - (\mathbf{u} \cdot \nabla) \mathbf{u} - \nabla p$, where $\mathbf{u}$ is a four-dimensional vector (three spatial dimensions and time), $x_p$ is the position of the particle, $\nu$ is the kinematic viscosity, $\nabla p$ is the pressure gradient, and $(\mathbf{u} \cdot \nabla) \mathbf{u}$ is convection [S2, unofficial]. Smaller viscosity leads to more rotation [S2, unofficial].

## Background
The specific implementations cited here draw from GPU Gems 3 [S2, unofficial] and Unity-based open-source projects [S1, unofficial]. Not given in the sources.

## Sources
[S1] PDF Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
[S2] Realistic Real-Time Rendering of Fire in a Production System, https://clas.iusb.edu/math-compsci/_prior-proposals/YVanzine_Proposal.pdf, fetched 2026-10-28 (unofficial)
