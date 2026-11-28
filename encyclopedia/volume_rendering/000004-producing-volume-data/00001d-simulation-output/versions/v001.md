# Simulation output

Also called: Simulation-based volume generation, Physics-driven volume data

## Overview
This family covers methods that generate volume data by solving physical equations (fluid dynamics, thermodynamics, or particle transport) rather than scanning physical objects or defining static mathematical fields. It solves the problem of creating temporally coherent, physically plausible volume attributes (density, temperature, velocity) for rendering. It differs from [Voxelization](12-voxelization.md) by deriving data from dynamic simulation rather than geometric sampling, and from [Procedural and implicit volumes](13-procedural-and-implicit-volumes.md) by relying on time-evolving state variables rather than static spatial functions.

## Shared principles
All methods in this family share the following mechanism:
1.  **State Representation**: The simulation state is stored in a discrete domain, typically a 3D grid (for Eulerian methods like smoke/fire) or a set of particles (for Lagrangian methods like PBF or massless particles).
2.  **Time Integration**: The state evolves over time steps $\Delta t$ by solving differential equations (e.g., Navier-Stokes, advection-diffusion, or constraint satisfaction).
3.  **Attribute Advection**: Scalar quantities (density, temperature, reaction coordinates) are transported by the velocity field, often using semi-Lagrangian schemes (e.g., MacCormack) or particle tracing.
4.  **Boundary Handling**: Interactions with domain boundaries (walls, obstacles) are enforced via masking, reflection, or constraint projection.
5.  **Output Generation**: The final simulation state (e.g., density field, particle positions) is converted into a format suitable for volume rendering (e.g., 3D textures, point clouds).

## Subtopics

### [Position Based Fluids](1765-position-based-fluids.md)
A particle-based fluid simulation method that enforces incompressibility by correcting particle positions based on density constraints, allowing real-time simulation without meshing. It differs from traditional Smoothed Particle Hydrodynamics (SPH) by solving for positions directly rather than pressures, which stabilizes the simulation and prevents density drift. Choose this when real-time, incompressible fluid behavior is required without the stability issues of pressure-based solvers.

### [Voxel-based 3D Fire Simulation](3411-voxel-based-3d-fire-simulation.md)
A simulation using a 3D voxelized grid to hold attributes: velocity, smoke density, temperature, and fuel level. It differs from general smoke solvers by explicitly tracking fuel and reaction coordinates to model fire extinction and propagation. Choose this when visual fidelity of fire dynamics (ignition, burning, extinction) is required.

### [Smoke Advection](4639-smoke-advection.md)
Tracking additional quantities like density and temperature to visualize smoke, advected by the fluid velocity using the same MacCormack routine as velocity. It differs from velocity-only solvers by maintaining scalar fields that influence rendering and physical forces (e.g., buoyancy). Choose this when smoke appearance and thermal effects are critical.

### Gaussian Splashing
A method that performs physical simulations using Position-Based Fluids. It reconstructs surfaces and disentangles objects for active simulation, using a mesh-based simulation framework with Gaussians attached to the meshes for texture rendering [S2, unofficial]. It differs from standard PBF by integrating the simulation directly into a 3D Gaussian Splatting rendering pipeline, allowing for physically accurate rendering of fluid surfaces without traditional mesh extraction. Choose this when using 3D Gaussian Splatting for real-time fluid rendering and surface reconstruction is desired.

### Smoke Source Injection
A Unity MonoBehaviour component that injects density and velocity into the Navier-Stokes solver at the transform's world position during FixedUpdate [S1, unofficial]. It differs from global source terms by providing localized, object-driven injection of smoke. Choose this when interactive, object-based smoke emission is required in a game engine context.

### Fire Reaction Coordinate
Tracking a reaction coordinate to simulate fire, which indicates time since ignition and fuel exhaustion [S3, unofficial]. The reaction coordinate $Y$ evolves as $\frac{\partial Y}{\partial t} = -(u \cdot \nabla) Y - k$, where $u$ is velocity and $k$ is a constant decrement per time step [S3, unofficial]. A value of 1 indicates just-ignited gas, and a value < 0 indicates fuel exhaustion [S3, unofficial]. It differs from simple density fields by explicitly modeling the fuel consumption process. Choose this when realistic fire behavior (ignition, burning, extinction) is needed.

### Buoyant Force for Smoke
Adding a buoyant force to the velocity field based on temperature differences to make hot smoke rise and cool smoke fall [S3, unofficial]. The buoyant force is calculated as $\mathbf{f}_{buoyancy} = \frac{Pmg}{R} \left( \frac{1}{T_0} - \frac{1}{T} \right) \mathbf{z}$, where $P$ is pressure, $m$ is molar mass, $g$ is gravity, $R$ is the universal gas constant, $T_0$ is ambient temperature, $T$ is advected temperature, and $\mathbf{z}$ is the normalized upward vector [S3, unofficial]. It differs from neutral buoyancy simulations by introducing temperature-dependent vertical acceleration. Choose this when realistic smoke rise and cooling effects are required.

### Massless Particle System
Particles are used as tracer elements to visualize flow, as droplets to model fire suppression, or as fuel elements (embers). Positions are determined by a differential equation assuming drag force is large compared to gravity [S4, unofficial]. Particle positions are determined using $\frac{dx_p}{dt} = V(x_p, t)$, where $x_p$ is particle position and $V$ is velocity [S4, unofficial]. It differs from dense particle fluids by treating particles as passive tracers or sparse embers rather than a continuous medium. Choose this for visualizing flow paths or sparse particulate effects like embers.

## Background
Simulation-based volume generation has its roots in computational fluid dynamics (CFD) and computer graphics research from the 1980s and 1990s. Jos Stam's stable fluids [G] laid the groundwork for Eulerian grid-based smoke simulation. Position-Based Fluids (PBF) was introduced by Müller et al. (2007) to address stability issues in SPH [S1, unofficial]. The integration of fire dynamics with reaction coordinates was popularized by Nguyen et al. (2002) in the context of GPU-based fluid simulation [S3, unofficial]. Gaussian Splashing represents a recent development (2024-2025) integrating physics with 3D Gaussian Splatting [S2, unofficial]. Massless particle systems have been used in fire safety visualization tools like Smokeview, developed by NIST, for decades [S4, unofficial]. These methods are widely used in visual effects, scientific visualization, and real-time game engines.

## Sources
[S1] furkanunsal42/VolumetricSmoke, https://github.com/furkanunsal42/VolumetricSmoke, fetched 2026-10-26 (unofficial)
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
[S3] PDF Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
[S4] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
