# Grid-Based Simulation

Also called: Eulerian Fluid Simulation, Grid-Based Fluid Dynamics

## Overview
Grid-Based Simulation maps physical sources (fluids, smoke, fire) into a discrete volumetric representation by solving partial differential equations on a structured spatial domain [S1, S2]. It differs from particle-based siblings like [Gaussian Splashing](1387-gaussian-splashing.md) and [Position Based Fluids](1765-position-based-fluids.md) by storing state in fixed voxel arrays, enabling efficient divergence and advection calculations via 3D textures [S1].

## Shared principles
All methods in this family operate on a structured 3D grid (voxelization) where physical quantities are stored at cell centers or faces [G]. The core mechanism involves:
1.  **Domain Definition**: A 3D array of resolution $(N_x, N_y, N_z)$ defines the spatial domain [G].
2.  **State Storage**: Scalar fields (density $\rho$, pressure $p$, temperature $T$) and vector fields (velocity $u$) are stored in dedicated textures or arrays [S1, S2].
3.  **Navier-Stokes Solver**: Most methods solve the incompressible Navier-Stokes equations to propagate velocity, typically using a projection method (advection, diffusion, force application, pressure projection) [S1].
4.  **Advection**: Transport of attributes (smoke, fuel, velocity) is performed using semi-Lagrangian tracing or finite differences [S1].
5.  **Rendering Interface**: The resulting grid serves as a volumetric texture container for ray marching or voxel rasterization [S1, S2].

## Subtopics

### [Voxel-based 3D Fire Simulation](3411-voxel-based-3d-fire-simulation.md)
Simulates fire dynamics with specific buoyancy and vorticity forces on a $128^3$ grid. Chosen when fire-specific physics (curling, rising) are required.

### [GPU-based Fluid Simulation with Voxel Rendering](7400-gpu-based-fluid-simulation-with-voxel-rendering.md)
Integrates real-time incompressible fluid simulation with sparse voxel rasterization using Direct3D geometry shaders. Chosen for efficient GPU-only visualization of general fluids.

### [Niagara Fluids Volume Rendering](7739-niagara-fluids-volume-rendering.md)
Repurposes the Niagara grid system as a passive texture container, ignoring physics updates. Chosen for interactive volume rendering within Unreal Engine, despite high memory overhead.

## Background
Grid-based fluid simulation originates from the Eulerian approach to computational fluid dynamics (CFD) [G]. It has been adapted for real-time graphics to simulate smoke, fire, and water by leveraging GPU compute shaders for the Navier-Stokes solver [S1].

## Sources
[S1] GPU-based Fluid Simulation with Voxel Rendering, https://github.com/ashawkey/voxel-fluid-simulation, 2024-05-20
[S2] Niagara Fluids Volume Rendering, https://github.com/UnrealEngine/Engine, 2024-05-20
