# PAC-NeRF

Also called: Physics Augmented Continuum Neural Radiance Fields

## Overview
PAC-NeRF is a method for geometry-agnostic system identification that estimates the unknown geometry and physical parameters of highly dynamic objects from multi-view videos [S1, unofficial]. It solves the problem of recovering physical properties (e.g., Young’s modulus, viscosity) without assuming prior object geometry or topology [S1, unofficial]. It differs from standard dynamic NeRFs by enforcing continuum mechanics conservation laws via a differentiable Material Point Method (MPM) simulator, rather than learning dynamics from scratch [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of posed multi-view video frames $\{\hat{C}(r, t_i)\}$ of a dynamic scene.
*   **Output:** An explicit geometric representation (voxel grid) and physical parameters (e.g., material constants) that minimize the difference between rendered and observed images.

**Data Structures:**
The method uses a hybrid Eulerian-Lagrangian representation [S1, unofficial]:
1.  **Eulerian Grid:** Stores the NeRF fields.
    *   **Density Grid ($G_\sigma$):** Stores volume density $\sigma(x, t)$. Layout: $\mathbb{R}^{I \times J \times K}$ [S1, unofficial].
    *   **Appearance Grid ($G_c$):** Stores latent features for color prediction. Layout: $\mathbb{R}^{I \times J \times K \times P}$, where $P$ is the number of feature channels [S1, unofficial].
2.  **Lagrangian Particles:** A set of particles that advect the field values. Each particle carries state information used in the MPM simulation [S1, unofficial].

**Query Mechanism:**
For a query point $p$ in world space at time $t$, grid values are retrieved via trilinear interpolation [S1, unofficial]:
$$ f(p) = \text{trilinear\_interp}(p, G) $$
This yields scalar density $\sigma(p, t)$ and feature vector $f(p, t)$ [S1, unofficial].

**Decoding:**
The feature vector $f(p, t)$ and viewing direction $\omega$ are passed to a decoding MLP $\mathcal{S}$ to predict color $\mathbf{c}$ [S1, unofficial]:
$$ \mathbf{c} = \mathcal{S}(f(p, t), \omega) $$

**Simulation and Advection (MPM):**
The core innovation is the enforcement of physical conservation laws using the Material Point Method (MPM) [S1, unofficial]:
1.  **Material Derivative:** The density $\sigma$ and color field $c$ are assumed to be conserved along particle trajectories:
    $$ \frac{D\sigma}{Dt} = 0, \quad \frac{Dc}{Dt} = 0 $$
    where $\frac{D\phi}{Dt} = \frac{\partial \phi}{\partial t} + \mathbf{v} \cdot \nabla \phi$ and $\mathbf{v}$ is the velocity field [S1, unofficial].
2.  **Momentum Conservation:** The velocity field evolves according to:
    $$ \frac{D\mathbf{v}}{Dt} = \frac{1}{\rho} \nabla \cdot \mathbf{T} + \mathbf{g} $$
    where $\rho$ is physical density, $\mathbf{T}$ is the internal Cauchy stress tensor, and $\mathbf{g}$ is gravity [S1, unofficial].
3.  **Differentiable Physics:** The MPM simulator computes the velocity field $\mathbf{v}$ and updates particle positions. The stress tensor $\mathbf{T}$ is computed based on physical parameters (e.g., Young’s modulus) [S1, unofficial].

**Optimization Loop:**
1.  Initialize Eulerian voxel NeRF for density $\sigma$ and color $c$ [S1, unofficial].
2.  Bind renderable quantities to Lagrangian particles via a sampling scheme [S1, unofficial].
3.  Simulate particle dynamics using differentiable MPM [S1, unofficial].
4.  Convert particle states back to the Eulerian grid via Grid-to-Particle (G2P) and Particle-to-Grid (P2G) transforms [S1, unofficial].
5.  Render images from the Eulerian voxel NeRF using the volume rendering integral [S1, unofficial]:
    $$ C(r, t) = \int_{s_n}^{s_f} T(s, t) \sigma(r(s), t) c(r(s), \omega, t) ds + c_{bg} T(s_f, t) $$
    where $T(s, t) = \exp\left(-\int_{s_n}^{s} \sigma(r(\bar{s}), t) d\bar{s}\right)$ [S1, unofficial].
6.  Compute loss against ground truth video frames [S1, unofficial]:
    $$ L_{render} = \frac{1}{N} \sum_{i=0}^{N-1} \frac{1}{|\mathcal{R}|} \sum_{r \in \mathcal{R}} \| C(r, t_i) - \hat{C}(r, t_i) \|^2 $$
    where $N$ is the number of frames and $\mathcal{R}$ is the set of rays [S1, unofficial].
7.  Optimize geometry (grid values) and physical parameters via gradient-based optimization [S1, unofficial].

**Parameters:**
*   Physical parameters to estimate: Young’s modulus, fluid viscosity, friction angles, etc. [S1, unofficial].
*   Typical values: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Topology Changes:** Standard mesh-based methods fail here; PAC-NeRF uses MPM to naturally handle topology changes and frictional contacts [S1, unofficial].
*   **Unknown Geometry:** The method does not require watertight mesh representations or known object geometries [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Accelerates NeRF rendering by two orders of magnitude compared to MLP-based dynamic NeRFs due to the Eulerian voxel representation [S1, unofficial].
*   **Memory Complexity:** Depends on the voxel grid resolution ($I \times J \times K$) and the number of Lagrangian particles. Not given in the sources.
*   **Measured Performance:** Validated on elastic bodies, plasticine, sand, Newtonian and non-Newtonian fluids [S1, unofficial].
*   **Strengths:** Data-efficient, physically plausible, interpretable, handles complex dynamics and topology changes [S1, unofficial].
*   **Limitations:** Requires multi-view video input; relies on the accuracy of the differentiable MPM simulator [S1, unofficial].
*   **When to use:** When estimating physical properties of dynamic objects from video without prior geometric knowledge.
*   **When not to use:** When single-view input is available or when the scene does not involve continuum mechanics (e.g., rigid body kinematics only).

## Background
PAC-NeRF was introduced to address the challenge of inferring geometric and physical properties from visual observations without assuming object geometry [S1, unofficial]. It builds on the concept of Neural Radiance Fields (NeRF) [S1, unofficial] and the Material Point Method (MPM) for physics simulation [S1, unofficial]. It differs from D-NeRF and other dynamic NeRFs by using a prescriptive physics model rather than learning dynamics from scratch [S1, unofficial]. It also differs from existing system identification methods by not requiring known object geometries [S1, unofficial].

## Sources
[S1] PAC-NeRF: Physics Augmented Continuum Neural Radiance Fields for Geometry-Agnostic System Identification, https://arxiv.org/abs/2303.05512v1, fetched 2026-10-26 (unofficial)
