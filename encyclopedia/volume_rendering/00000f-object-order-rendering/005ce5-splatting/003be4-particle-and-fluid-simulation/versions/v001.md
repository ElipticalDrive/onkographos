# Particle and Fluid Simulation

Also called: Particle-based Fluid Rendering, Particle Splatting for Fluids, Hybrid Particle-Grid Water

## Overview
Particle and Fluid Simulation methods render dynamic fluids and participating media by treating them as collections of discrete particles (often Gaussian kernels or billboards) that are splatted onto the image plane [S1, S2, S3]. This approach solves the problem of efficiently visualizing complex, time-evolving fluid dynamics (such as splashes, foam, and spray) and gaseous volumes without the memory overhead of dense volumetric grids [S1, S2]. It sets itself apart from grid-based volume rendering by leveraging the flexibility of particle systems to handle free surfaces and topological changes, while maintaining real-time performance through hardware-accelerated splatting [S1, S2].

## Shared principles
The Particle and Fluid Simulation family shares the fundamental mechanism of decomposing a continuous fluid or medium into a set of discrete particles, each carrying physical properties (position, velocity, density) and visual attributes (color, opacity, normal) [S1, S2, S3]. These particles are transformed into screen space and rasterized using splatting techniques, accumulating their contributions to form the final image [S1, S2, S3].

1.  **Particle Representation:** The fluid or medium is represented by a set of particles $P_i$. Each particle $i$ is defined by its position $\mathbf{p}_i$, velocity $\mathbf{v}_i$, and visual parameters such as radius $r_i$, color $\mathbf{c}_i$, and opacity $\alpha_i$ [S1, S2, S3]. In Gaussian splatting variants, particles are modeled as 3D Gaussians with a mean position $\mu_i$ and a covariance matrix $\Sigma_i$ [S1].
2.  **Simulation and Update:** Particle positions and properties are updated over time using physics-based simulations, such as Smoothed Particle Hydrodynamics (SPH), Position-Based Dynamics (PBD), or grid-coupled methods [S1, S3]. For example, in PBF, constraints are solved to maintain incompressibility and surface tension [S1].
3.  **Splatting and Projection:** Each particle is projected onto the 2D image plane. For Gaussian particles, the covariance matrix is transformed by the view and projection matrices to determine the 2D footprint (ellipsoid) on the screen [S1]. The contribution of each particle to a pixel is calculated based on its kernel function (e.g., Gaussian) and its depth [S1, S2].
4.  **Compositing:** Particle contributions are accumulated to form the final image. Standard alpha blending is used for transparency [S1, S2]. However, for specific effects like fluid thickness or foam intensity, additive blending may be employed to accumulate density or intensity values before final compositing [S1, S3].
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

## Background
Particle-based fluid simulation and rendering has evolved from simple billboard techniques for smoke and clouds to sophisticated coupled particle-grid systems for realistic water and foam [S1, S2, S3]. Early methods focused on billboarding for performance in games and simulations [S1, S2]. The introduction of Smoothed Particle Hydrodynamics (SPH) and Position-Based Dynamics (PBD) allowed for more physically accurate fluid simulations [S1]. Recent advancements, such as Gaussian Splashing, integrate these particle simulations with modern neural rendering techniques like 3D Gaussian Splatting to achieve high-fidelity, real-time rendering of dynamic fluids with complex optical effects like refraction and foam [S1]. The particle map technique, introduced by Adabala and Manohar, provides a framework for efficient light transport in participating media using particle-based structures [S2]. Hybrid approaches, combining particle simulations with grid-based velocity fields, offer a balance between physical accuracy and computational efficiency for large-scale water simulations [S3].

## Sources
[S1] Gaussian Splashing: Unified Particles for Versatile Motion Synthesis and Rendering, https://doi.org/10.1109/CVPR52734.2025.00057, fetched 2026-10-29
[S2] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
[S3] Part4 Chapter16 Fluid Simulation Demo, https://github.com/Razria93/Graphics_Study_Archive/issues/33, fetched 2026-10-26
