# Particle and Fluid Simulation


## Overview
Particle and Fluid Simulation methods render dynamic fluids and participating media by treating them as collections of discrete particles that are splatted onto the image plane [S1, S2, S3]. This approach solves the problem of efficiently visualizing complex, time-evolving fluid dynamics and gaseous volumes without the memory overhead of dense volumetric grids [S1, S2]. It sets itself apart from its siblings like [Billboards and particles](9045-billboards-and-particles.md) and [Screen-Space Fluid Rendering Pipeline](14414-screen-space-fluid-rendering-pipeline.md) by explicitly simulating physical particle properties (position, velocity, density) and projecting them as splats, rather than relying solely on pre-defined textures or screen-space thickness accumulation [S1, S2, S3].

## Shared principles
The Particle and Fluid Simulation family shares the fundamental mechanism of decomposing a continuous fluid or medium into a set of discrete particles, each carrying physical properties (position, velocity, density) and visual attributes (color, opacity, normal) [S1, S2, S3]. These particles are transformed into screen space and rasterized using splatting techniques, accumulating their contributions to form the final image [G].

1.  **Particle Representation:** The fluid or medium is represented by a set of particles $P_i$ [S1, S2, S3]. Each particle $i$ is defined by its position $\mathbf{p}_i$, velocity $\mathbf{v}_i$, and visual parameters such as radius $r_i$, color $\mathbf{c}_i$, and opacity $\alpha_i$ [S1][S4, unofficial]. In Gaussian splatting variants, particles are modeled as 3D Gaussians with a mean position and a covariance matrix $\Sigma_i$ [S1].
2.  **Simulation and Update:** Particle positions and properties are updated over time using physics-based simulations, such as Smoothed Particle Hydrodynamics (SPH), Position-Based Dynamics (PBD), or grid-coupled methods [S1, S3]. For example, in PBF, constraints are solved to maintain incompressibility and surface tension [S1].
3.  **Splatting and Projection:** Each particle is projected onto the 2D image plane [G]. For Gaussian particles, the covariance matrix is transformed by the view and projection matrices to determine the 2D footprint (ellipsoid) on the screen [S1]. The contribution of each particle to a pixel is calculated based on its kernel function [S1].
4.  **Compositing:** Particle contributions are accumulated to form the final image [G]. Standard alpha blending is used for transparency [S1, S2]. However, for specific effects like fluid thickness or foam intensity, additive blending may be employed to accumulate density or intensity values before final compositing [S1, S3].
5.  **Hybrid Approaches:** Some methods combine particle simulations with grid-based velocity projections to improve stability and performance, switching between density volume display and Signed-Distance Field (SDF) raycasting for rendering [S3].

## Subtopics

### [Billboards and particles](9045-billboards-and-particles.md)
Billboards and particles are a method for approximating participating media effects or complex geometry (such as forests) using 2D semi-transparent textures that always face the camera. This approach is primarily chosen for its low computational cost, making it suitable for real-time rendering in industrial simulators and video games where performance is critical. Unlike higher-fidelity volume rendering techniques, it sacrifices geometric accuracy and parallax richness for speed, relying on pre-defined textures or rule-based particle systems.

### Particle-based Foam Rendering
#### definition
- [S1] Rendering of spray, foam, and bubble particles advected by fluid, using modified additive splatting with specific kernels and intensities.
- [S1] A post-processing step to enhance fluid realism by synthesizing spray, foam, and bubble particles.
#### step
- [S1] Splat particles into foam intensity image using modified additive splatting. Use different kernels for different particle types.
- [S1] Apply a curve to the foam intensity image to scale it into [0, 1] and compose it with rendering.
- [S1] Generate fluid-air mixtures at the crest of the wave.
- [S1] Generate fluid-air mixtures in the impact zone.
#### parameter
- [S1] Foam particle intensity: Larger overall intensity to increase visibility.
- [S1] Spray particle intensity: Smaller intensity to make them less prominent.
- [S1] Bubble particle kernel: Hollow circle structure for convincing underwater appearance.
- [S1] Kernel radius: Typically 2 pixels, scaled based on particle depth in view.

### Fluid Rendering with Refraction
#### definition
- [S1] Renders dynamic particle-based fluids using ellipsoids splatting, modeling reflection via PBR and refraction via thickness-dependent diffuse color using Beer's Law.
#### step
- [S1] Generate spherical fluid Gaussian kernels at each fluid particle. Set initial covariance Ap by particle radius.
- [S1] Set normal np to the surface normal of the nearest surface fluid particle from PBF simulation.
- [S1] Compute fluid thickness τ from modified splatting pipeline using additive blending instead of alpha blending.
- [S1] Shade deformed Gaussian kernels with material {dp, sp, ρp} using Eq. (4) and splat into image cbg.
- [S1] Re-engineer nearly-soft shadows into the system.
- [S1] Combine cbg (solids/background) and cfluid (fluids) to achieve final rendering result.
#### formula
- [S1] dp = e^(-kτp) cbg_p. Where dp is the refraction color, k is absorption coefficient, τp is fluid thickness at pixel p, and cbg_p is the background color.
#### parameter
- [S1] sp = 1 (specular material intensity for fluids).
- [S1] ρp = 0.05 (roughness for fluids).

### Particle Map
#### definition
- [S2] A method using particle systems to model gaseous volumes, managing particles and illumination fields via a median k-dtree called a particle map, allowing inhomogeneous anisotropically scattering media and time evolution via fluid dynamics.
#### step
- [S2] Use a median k-dtree (particle map) to manage particles and their associated illumination field, storing information for associative searches.
- [S2] When a scattering event occurs, store related information in the particle map instead of a photon map.
- [S2] Distribute incident energy among particles near the point of interaction.
- [S2] Store only the energy that would be scattered into the viewing direction, sacrificing viewpoint independence to avoid the prohibitive cost of storing incident direction for nearby particles.
- [S2] Solve the integral transport equation and source radiance using particle maps via nearest neighbor queries.

### Hybrid Particle-Grid Water
#### step
- [S3] Combine particle simulation with grid velocity projection.
- [S3] After grid projection and particle step, sort particles by cell index.
- [S3] Create density, velocity, and signed-distance field (SDF) using weighted contribution of adjacent particles.
- [S3] Toggle rendering between density volume display and SDF raycasting display.

### Particle Advection and Rendering
#### definition
- [S4, unofficial] Displaying movement in time-dependent data using GPU-computed particles emitted from user-placed probe volumes.
#### step
- [S4, unofficial] Advect particles during their lifetime based on the values of the vector field.
- [S4, unofficial] Render particles as point primitives at their calculated positions (simplest method).
- [S4, unofficial] Render particles as billboards using a geometry shader to create quads around particle positions.
- [S4, unofficial] Calculate a sphere based on camera distance and position to perform lighting calculations on the billboards.
#### parameter
- [S4, unofficial] User-controllable parameters: particle size, color, lifetime, and amount.

### Camera-Following Particle System
#### step
- [S5, unofficial] Per-archetype weather particles spawn in a camera-following box, fall/drift, and wrap. Examples: Mars red dust storm, Titan methane drizzle, Io ash embers (additive), icy-body snow/crystals, faint Earth motes. Airless regolith bodies get none. Count scales with a quality budget.

## Background
Particle-based fluid simulation and rendering has evolved from simple billboard techniques for smoke and clouds to sophisticated coupled particle-grid systems for realistic water and foam [S1, S2, S3]. The particle map technique, introduced by Adabala and Manohar, provides a framework for efficient light transport in participating media using particle-based structures [S2]. Smoothed Particle Hydrodynamics (SPH) and Position-Based Dynamics (PBD) enabled physically accurate fluid simulations [S1]. Recent advancements, such as Gaussian Splashing, integrate these particle simulations with modern neural rendering techniques like 3D Gaussian Splatting to achieve high-fidelity, real-time rendering of dynamic fluids with complex optical effects like refraction and foam [S1]. Hybrid approaches, combining particle simulations with grid-based velocity fields, offer a balance between physical accuracy and computational efficiency for large-scale water simulations [S3]. GPU-computed particle advection allows interactive visualization of vector fields in scientific data analysis [S4, unofficial]. Camera-following particle systems are utilized in procedural planetary exploration to simulate atmospheric phenomena like dust storms and snow [S5, unofficial].

## Sources
[S1] Gaussian Splashing: Unified Particles for Versatile Motion Synthesis and Rendering, https://doi.org/10.1109/CVPR52734.2025.00057, fetched 2026-10-29
[S2] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
[S3] Part4 Chapter16 Fluid Simulation Demo, https://github.com/Razria93/Graphics_Study_Archive/issues/33, fetched 2026-10-26
[S4] Interactive Visual data analysis | Daniel Suttor, https://dasutt.github.io/stories/interactive-visual-data-analysis/, fetched 2026-10-28 (unofficial)
[S5] Phase 9: Voxel Surface Exploration, https://github.com/LundstedtAdam/SolarSystem/pull/4, fetched 2026-10-28 (unofficial)
