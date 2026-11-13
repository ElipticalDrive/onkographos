# Simulation output
## Overview
This family covers methods that generate volumetric data by simulating physical phenomena (fluids, fire, smoke) or kinematic systems, producing a discrete or analytic representation suitable for rendering [G]. Unlike [Voxelization](12-voxelization.md) which converts static geometry, or [Procedural and implicit volumes](13-procedural-and-implicit-volumes.md) which define volume via analytic functions, simulation output methods evolve state over time based on physical laws (e.g., Navier-Stokes, Position-Based Dynamics) or kinematic rules [G]. The core mechanism involves defining a spatial domain (grid or particles), evolving physical quantities (velocity, density, temperature) through time-stepping, and storing the result in a structure (voxel grid, particle list, or Gaussian kernel) that serves as the input for a volume renderer [G].

## Shared principles
All methods in this family map a conceptual or physical source into a discrete or analytic volumetric representation suitable for rendering [G]. The core mechanism involves:
1. **Domain Definition**: Establishing a spatial domain, typically a 3D grid with resolution $(N_x, N_y, N_z)$ and bounding box, or an infinite analytic space [G].
2. **Value Assignment**: Determining the value (density, distance, color) for each point in the domain based on the source method (geometric intersection, function evaluation, or physical state) [G].
3. **Storage/Access**: Either storing the result in a discrete array (voxelization, simulation) or providing an interface to compute it on-the-fly (procedural) [G].

## Subtopics

### [Gaussian Splashing](1387-gaussian-splashing.md)
Gaussian Splashing combines 3D Gaussian Splatting with Position-Based Dynamics to simulate fluids, deformable bodies, and rigid objects. It distinguishes itself from vanilla 3DGS by enriching Gaussian kernels with material parameters and aligning kernel orientation with surface normals to eliminate spiky noise. Choose this when interactive, physically-based rendering of deformable objects from novel views is required [G].

### [Position Based Fluids](1765-position-based-fluids.md)
Position Based Fluids is a Lagrangian particle-based method that enforces incompressibility by iteratively correcting particle positions based on density constraints. It distinguishes itself from Smoothed Particle Hydrodynamics by treating fluid dynamics as a constraint satisfaction problem rather than solving Navier-Stokes equations via pressure gradients, offering superior stability for incompressible flows. Choose this for real-time, stable fluid simulation where meshing is undesirable [G].

### [Navier-Stokes Solver Components](13501-navier-stokes-solver-components.md)
This component provides the mathematical and algorithmic building blocks for solving the incompressible Navier-Stokes equations on a grid. It distinguishes itself by modularizing advection, diffusion, and projection steps, allowing flexible integration into larger simulation pipelines. Choose this when building a custom grid-based fluid solver from scratch [G].

### [Grid-Based Simulation](13502-grid-based-simulation.md)
Grid-Based Simulation maps physical sources (fluids, smoke, fire) into a discrete volumetric representation by solving partial differential equations on a structured spatial domain. It differs from particle-based siblings by storing state in fixed voxel arrays, enabling efficient divergence and advection calculations via 3D textures. Choose this for grid-based fire and smoke effects with explicit physical coupling [G].

#### Fire Reaction Coordinate
A method tracking a reaction coordinate to simulate fire, indicating time since ignition and fuel exhaustion [S2, unofficial]. The coordinate $Y$ evolves as $\frac{\partial Y}{\partial t} = -(u \cdot \nabla)Y - k$, where $k$ is a constant decrement per time step [S2, unofficial]. A value of 1 indicates ignition, and $<0$ indicates exhaustion [S2, unofficial]. It is advected using `PS_ADVECT_MACCORMACK` with $k$ added to the result, and a spherical region is set to 1 to continuously generate fuel [S2, unofficial].

#### Massless Particle System
Particles are used as tracer elements to visualize flow, droplets for fire suppression, or fuel elements (embers) [S3, unofficial]. Positions are determined by $\frac{dx_p}{dt} = V(x_p, t)$, assuming drag force is large compared to gravity ($mg$) [S3, unofficial]. Particle size is specified via `glPointSize(partpointsize)` and streak line length in seconds [S3, unofficial]. Velocity field and particle motion are computed by FDS and communicated to Smokeview via data files [S3, unofficial].

#### Galaxy Kinematic Simulation
A kinematic approach to simulate galaxy time evolution using differential rotation laws and first-order integration, without full dynamic simulation [S4]. Gas and stars on the disk rotate according to a typical galaxy rotation curve, while bulge stars rotate with constant velocity balancing gravitational attraction [S4]. Equations of motion are integrated using first-order methods; numerical errors amplify, making the simulation meaningful for only a few hundred million years [S4]. Double precision is required for geometrical calculations to handle the large dynamic range [S4]. Used to render the galaxy component within cosmological simulations [S4].

#### Particle Simulation for Fire Dynamics
A simulation driving the control points of the deformation curve, accounting for wind, high-energy dispersion, and thermal buoyancy [S5]. The velocity $\vec{v}$ combines base fire velocity, wind $\vec{w}(x,t)$, high-energy dispersion $\vec{d}(T)$, and thermal buoyancy $\vec{c}(T, \text{age})$ [S5]. Thermal expansion uses $\beta = 1/T$, gravity $g$, and ambient temperature $T_0 \approx 300\text{K}$ [S5].

## Background
Simulation-based volume data generation has roots in computational fluid dynamics (CFD) and scientific visualization [G]. Early methods focused on grid-based Eulerian approaches for smoke and fire, such as the MacCormack advection scheme and reaction coordinate tracking described in GPU Gems [S2, unofficial]. Lagrangian methods like Position-Based Fluids emerged to address stability issues in particle-based simulations [S2, unofficial]. In scientific visualization, tools like Smokeview used massless particles and volumetric rendering to visualize Fire Dynamics Simulation (FDS) data [S3, unofficial]. Kinematic simulations have been applied in astrophysics to visualize galaxy evolution within cosmological contexts using double-precision arithmetic [S4]. Procedural fire systems also integrate particle simulations for macro-level effects like wind and thermal buoyancy, combined with pixel-shader noise for micro-level flicker [S5].

## Sources
[S1] furkanunsal42/VolumetricSmoke, https://github.com/furkanunsal42/VolumetricSmoke, fetched 2026-10-26 (unofficial)
[S2] PDF Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
[S3] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
[S4] Splotch: visualizing cosmological simulations, https://doi.org/10.1088/1367-2630/10/12/125006, fetched 2026-10-28
[S5] Real-time procedural volumetric fire, https://doi.org/10.1145/1230100.1230131, fetched 2026-11-05
