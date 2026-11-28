# Producing volume data
Also called: Volume data generation, Volume synthesis

## Overview
This family encompasses methods for generating the scalar or vector fields required for volume rendering, solving the problem of how to obtain or construct the input data structure (voxel grids, implicit fields, or simulation states) from continuous geometry, physical laws, or mathematical definitions. It is distinguished from [Image-order rendering](16-image-order-rendering.md) and [Object-order rendering](22-object-order-rendering.md) by focusing on data creation rather than pixel projection, and from [Acquisition](14-acquisition.md) by including computational synthesis rather than just physical sensing.

## Shared principles
All methods in this family map a conceptual or physical source into a discrete or analytic volumetric representation suitable for rendering. The core mechanism involves:
1.  **Domain Definition**: Establishing a spatial domain, typically a 3D grid with resolution $(N_x, N_y, N_z)$ and bounding box, or an infinite analytic space [S1, S3, unofficial].
2.  **Value Assignment**: Determining the value (density, distance, color) for each point in the domain based on the source method (geometric intersection, function evaluation, or physical state) [S1, S12].
3.  **Storage/Access**: Either storing the result in a discrete array (voxelization, simulation) or providing an interface to compute it on-the-fly (procedural) [S1, S3, unofficial].

## Subtopics

### [Voxelization](12-voxelization.md)
Converts continuous geometry (meshes, points) into a discrete 3D occupancy grid or scalar field. Choose this when you need a fixed-resolution, explicit representation for ray marching or collision detection. It differs from procedural volumes by being explicit and sampled, and from acquisition by being a computational conversion.

### [Procedural and implicit volumes](13-procedural-and-implicit-volumes.md)
Generates volumetric data on-the-fly using mathematical functions (e.g., noise, distance fields). Choose this for infinite resolution, memory efficiency, and organic shapes. It differs from voxelization by replacing storage with computation, and from simulation by being deterministic and static (unless animated via domain warping).

### [Acquisition](14-acquisition.md)
Generates volume data from physical sensing (CT, MRI, microscopy, seismic). Choose this when real-world data is required. It differs from computational methods by relying on physical measurement rather than algorithmic generation.

### [Simulation output](15-simulation-output.md)
Generates volume data by solving physical equations (fluid dynamics, thermodynamics). Choose this for physically plausible, temporally coherent attributes like smoke or fire. It differs from procedural volumes by relying on time-evolving state variables and from voxelization by deriving data from dynamic simulation rather than geometric sampling.

## Background
The generation of volume data is fundamental to computer graphics and scientific visualization. Early methods focused on voxelization of polygonal models for solid modeling and ray casting [S1]. Procedural volumes were popularized by Ken Perlin's noise functions and fractal Brownian motion in the 1980s for generating natural textures [S5, S6]. Simulation-based volumes emerged with the work of Jos Stam and others on stable fluids, enabling real-time rendering of dynamic phenomena [S1, S2]. Acquisition techniques have evolved from medical imaging (CT/MRI) to modern light field and volumetric video capture [S1, S3, unofficial].

## Sources
[S1] Volume rendering fundamentals, https://example.com/vol-rendering, 2023-10-01
[S2] Stable fluids and simulation, https://example.com/stable-fluids, 2023-10-01
[S3] Voxelization techniques, https://example.com/voxelization, 2023-10-01
[S5] Procedural noise and fractals, https://example.com/procedural-noise, 2023-10-01
[S6] Fractal Brownian Motion, https://example.com/fbm, 2023-10-01
[S7] Implicit surfaces and distance fields, https://example.com/implicit-surfaces, 2023-10-01
