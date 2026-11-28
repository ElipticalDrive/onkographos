# Particle-Based Simulation


## Overview
Particle-Based Simulation represents volumetric phenomena (fluids, smoke, fire) as a collection of discrete entities moving through space, solving for their trajectories rather than solving field equations on a grid [G]. It distinguishes itself from grid-based siblings like [Navier-Stokes Solvers](17619-navier-stokes-solvers.md) by decoupling the representation of matter from the computational domain, allowing for natural handling of topology changes and free surfaces without meshing or complex level-set management [G].

## Shared principles
See [Simulation output](15-simulation-output.md) for the family's shared principles.

## Subtopics

### [Position Based Fluids](1765-position-based-fluids.md)
A Lagrangian particle-based fluid simulation method that enforces incompressibility by correcting particle positions based on density constraints, allowing real-time simulation without meshing [G]. It distinguishes itself from Smoothed Particle Hydrodynamics (SPH) by treating fluid dynamics as a constraint satisfaction problem solved via iterative position projections rather than solving the Navier-Stokes equations through pressure gradients [G]. This approach provides superior stability for incompressible flows and simplifies the handling of surface tension and boundary conditions [G]. Choose it when stability and explicit constraint handling are prioritized over strict physical pressure derivation [G].

### [Massless Particle System](3970-massless-particle-system.md)
A visualization technique that represents fluid flow, smoke, or fire by tracking tracer particles advected by an underlying velocity field [G]. It solves the problem of visualizing complex flow patterns where static fields are insufficient, relying on the assumption that drag forces dominate gravitational forces on the tracers [G]. Unlike grid-based methods that compute volumetric density directly, this method generates discrete trajectories that can be rendered as points or streaks to reveal motion dynamics [G]. Choose it for post-processing flow field visualization rather than interactive simulation [G].

### [Lennard-James Potential Cloud Simulation](19871-lennard-james-potential-cloud-simulation.md)
Simulates clouds using particles by approximating the force between these particles based on the Lennard-James potential [G]. Choose it when modeling inter-particle forces that mimic molecular attraction and repulsion for volumetric cloud rendering [G].

### Particle Simulation for Fire Dynamics
A simulation driving the control points of the deformation curve, accounting for wind, high-energy dispersion, and thermal buoyancy [S1]. The velocity of a fire particle is determined by the sum of its base velocity, wind, dispersion, and buoyancy effects [S1]. The parameters include: $T$ (temperature of the fire), $\vec{v}$ (velocity of the fire), $\vec{w}()$ (wind velocity as a function of position and time), $\vec{d}()$ (velocity attributed to high energy dispersion as a function of temperature), $\vec{c}()$ (velocity attributed to thermal buoyancy as a function of the temperature and particle age), $\beta$ (thermal expansion coefficient of an ideal gas, $= 1/T$), $g$ (gravitational constant), and $T_0$ (environment's ambient temperature, usually about 300K) [S1].

## Background
Particle-based methods originated in astrophysics and hydrodynamics, notably Smoothed Particle Hydrodynamics (SPH) [G]. In computer graphics, they gained prominence for real-time fluid and fire simulation due to their ability to handle large deformations and free surfaces efficiently [G]. Position Based Dynamics (PBD) and its fluid variant (PBF) were introduced to address the instability and complexity of traditional SPH in real-time applications [G]. Massless particle systems are widely used for visualizing flow fields derived from other simulation methods [G].

## Sources
[S1] Real-time procedural volumetric fire, https://doi.org/10.1145/1230100.1230131, fetched 2026-11-05
