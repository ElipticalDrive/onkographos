# PhysGaussian
Also called: MPM over Gaussians
## Overview
PhysGaussian couples 3D Gaussian Splatting (3DGS) with continuum mechanics, specifically the Material Point Method (MPM), to enable the simulation of physical dynamics such as deformation, fracture, and fluid flow [S1, unofficial]. It provides a unified representation for both high-fidelity rendering and physics simulation, supporting materials like cloth, jelly, sand, and snow [S2, unofficial]. Unlike variants that convert Gaussians to meshes for simulation (e.g., GASP), PhysGaussian applies MPM directly over the Gaussian primitives, allowing for differentiable physical interactions within the Gaussian representation [S1, unofficial][S2, unofficial].

## How it works
PhysGaussian extends the standard 3DGS parameterization by assigning physical properties to each Gaussian. While the exact mathematical formulation for the coupling between the Gaussian attributes (position, covariance, opacity, SH coefficients) and the MPM state variables (mass, velocity, deformation gradient) is not detailed in the provided sources, the method operates by treating each Gaussian as a material point or a collection of points within the MPM grid [S1, unofficial][S2, unofficial].

1.  **Initialization**: Start with a standard 3DGS scene initialized via [Gaussian Model Initialization](2349-gaussian-model-initialization.md). Each Gaussian $i$ is defined by its center $\mathbf{p}_i$, covariance matrix $\Sigma_i$, opacity $\alpha_i$, and spherical harmonic coefficients $\mathbf{c}_i$ [S1, unofficial].
2.  **Physical Property Assignment**: Assign physical properties to each Gaussian. In MPM, particles carry mass, velocity, and deformation history. For PhysGaussian, these properties are associated with the Gaussians [S1, unofficial].
3.  **Simulation Step**:
    *   Map Gaussian properties to a background grid (standard MPM procedure) [S2, unofficial].
    *   Perform physics simulation (advection, external forces, constitutive model updates) on the grid [S2, unofficial].
    *   Map the updated grid velocities and positions back to the Gaussians [S2, unofficial].
4.  **Rendering**: Render the updated Gaussians using the standard differentiable rasterization pipeline [S1, unofficial].

*Note: The provided sources do not contain the specific equations for the mass assignment, force integration, or the re-parameterization of the Gaussian covariance after deformation. Implementation requires deriving these mappings from standard MPM literature applied to Gaussian primitives [G].*

## Subtopics

### [GASP](16066-gasp.md)
GASP is a pipeline that integrates Gaussian Splatting with physics-based simulations by converting flat Gaussians into triangle faces using the GaMeS reparametrization, simulating the vertices using a standard Material Point Method (MPM) engine, and reassembling the deformed triangles back into Gaussians. It solves the problem of applying physics to Gaussian representations without the artifacts caused by direct vertex processing or the incompatibility of 2DGS with triangle-based simulation. Unlike PhysGaussian, which uses MPM directly over Gaussians, GASP explicitly converts Gaussians to a mesh, simulates the mesh, and converts back, ensuring stable simulations for flat Gaussian components.

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Performance**: Not given in the sources.
- **Strengths**: Provides a single representation for both rendering and simulation, enabling applications in AR/VFX and robotics simulation where physical plausibility is required [S2, unofficial].
- **Limitations**: Not given in the sources.
- **When to use**: When physical dynamics (deformation, fracture, fluids) need to be simulated and rendered simultaneously from a 3DGS scan [S1, unofficial][S2, unofficial].
- **When not to use**: When only static rendering is needed, or when mesh-based simulation is preferred for stability with flat structures [S1, unofficial].

## Background
PhysGaussian was introduced in CVPR'24 [S2, unofficial]. It addresses the limitation of standard 3DGS, which is static and lacks physical properties, by integrating it with the Material Point Method (MPM) [S1, unofficial]. This allows for the simulation of various materials including cloth, jelly, sand, and snow [S2, unofficial].

## Sources
[S1] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
