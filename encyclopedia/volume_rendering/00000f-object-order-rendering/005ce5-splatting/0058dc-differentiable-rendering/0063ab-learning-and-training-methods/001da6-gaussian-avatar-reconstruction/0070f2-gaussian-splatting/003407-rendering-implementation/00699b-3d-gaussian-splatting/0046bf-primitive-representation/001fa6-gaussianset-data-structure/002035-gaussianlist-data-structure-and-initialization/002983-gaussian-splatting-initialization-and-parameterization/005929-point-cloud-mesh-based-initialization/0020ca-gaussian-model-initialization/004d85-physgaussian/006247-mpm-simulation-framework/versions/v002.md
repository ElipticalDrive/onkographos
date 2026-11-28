# MPM Simulation Framework

Also called: Material Point Method

## Overview
The MPM Simulation Framework is a physics-based simulation method used within the PhysGaussian pipeline to model the large-deformation behavior of materials such as cloth, jelly, sand, and snow [S1, unofficial]. It serves as the physical prior for soft-body control, allowing shape changes to be computed via physical simulation rather than purely geometric deformation . It is distinguished from its siblings, such as [GASP](16066-gasp.md), by its support for a wide variety of constitutive and plasticity models within a unified continuum mechanics framework [S1, unofficial].

## How it works
The framework tracks the deformation gradient $\mathbf{F}$ on Material Point Method (MPM) particles to measure their distortion compared to their initial state [S1, unofficial]. Internal forces are derived from the Cauchy stress tensor $\mathbf{T}$, which is defined by the deformation gradient $\mathbf{F}$ and the chosen constitutive model [S1, unofficial]. For plastic materials, the deformation gradient is constrained within an elastic region; if it exceeds this region, a return mapping $Z$ is applied to pull the deformation gradient back onto the yield surface [S1, unofficial].

### Inputs and Outputs
*   **Inputs**: Initial particle positions, current particle velocities, time step $\Delta t$, material parameters (Lamé parameters $\mu, \lambda$, yield stress $\tau_Y$, viscosity $\eta$), and the constitutive model type.
*   **Outputs**: Updated particle positions, velocities, and deformation gradients $\mathbf{F}$ for the next time step.

### Data Structures
*   **Particle State**: Each particle stores:
    *   Position $\mathbf{x}$ (3D vector).
    *   Velocity $\mathbf{v}$ (3D vector).
    *   Deformation Gradient $\mathbf{F}$ (3x3 matrix).
    *   Mass $m$ (scalar).
    *   Volume $V$ (scalar).

### Steps
1.  **Initialization**: Initialize particles with initial positions $\mathbf{x}_0$, velocities $\mathbf{v}_0$, and deformation gradient $\mathbf{F} = \mathbf{I}$ (identity matrix) [S1, unofficial].
2.  **Deformation Gradient Update**: For each particle, update the deformation gradient $\mathbf{F}$ based on the velocity gradient and time step $\Delta t$ [S1, unofficial].
3.  **Stress Calculation**: Compute the Cauchy stress tensor $\mathbf{T}$ using the current $\mathbf{F}$ and the selected constitutive model (e.g., Neo-Hookean, StVK, Viscoplastic) [S1, unofficial].
4.  **Plasticity Check**: For plastic models, check if the deformation gradient exceeds the elastic region using the specific yielding condition (e.g., von-Mises) [S1, unofficial].
5.  **Return Mapping**: If the yield condition is violated, apply the return mapping $Z(\mathbf{F})$ to project $\mathbf{F}$ back onto the yield surface [S1, unofficial].
6.  **Force Computation**: Compute internal forces from the stress tensor $\mathbf{T}$ and external forces (e.g., gravity) [S1, unofficial].
7.  **Integration**: Update particle velocities and positions using the computed forces and time step $\Delta t$ [S1, unofficial].

### Formulas
**Neo-Hookean Elasticity** [S1, unofficial]:
$$ \mathbf{T}(\mathbf{F}) = \mu(\mathbf{F}\mathbf{F}^T) + (\lambda \log(J) - \mu)\mathbf{I} $$
where $J = \det(\mathbf{F})$, and $\mu, \lambda$ are Lamé parameters related to Young’s modulus $E$ and Poisson’s ratio $\nu$ by:
$$ \mu = \frac{E}{2(1 + \nu)}, \quad \lambda = \frac{\nu E}{(1 + \nu)(1 - 2\nu)} $$

**StVK + von-Mises Plasticity** [S1, unofficial]:
$$ \mathbf{T}(\mathbf{F}) = \mathbf{U}(2\mu\mathbf{\epsilon} + \lambda \text{Tr}(\mathbf{\epsilon}))\mathbf{U}^T $$
where $\mathbf{F} = \mathbf{U}\mathbf{\Sigma}\mathbf{V}$ is the SVD of $\mathbf{F}$, and $\mathbf{\epsilon} = \log(\mathbf{\Sigma})$ is the Hencky strain. The von-Mises yielding condition is:
$$ \delta\gamma = \|\hat{\mathbf{\epsilon}}\| - \frac{\tau_Y}{2\mu} > 0 $$
where $\hat{\mathbf{\epsilon}}$ is the normalized Hencky strain and $\tau_Y$ is the yield stress. The return mapping is:
$$ Z(\mathbf{F}) = \begin{cases} \mathbf{F} & \delta\gamma \le 0 \\ \mathbf{U} \exp\left(\hat{\mathbf{\epsilon}} - \delta\gamma \frac{\hat{\mathbf{\epsilon}}}{\|\hat{\mathbf{\epsilon}}\|}\right)\mathbf{V}^T & \text{otherwise} \end{cases} $$

**Viscoplastic Model** [S1, unofficial]:
Define:
$$ \hat{\mu} = \frac{\text{Tr}(\mathbf{\Sigma}^2)}{d}, \quad \mathbf{s} = 2\hat{\mu}\hat{\mathbf{\epsilon}}, \quad \hat{s} = \|\mathbf{s}\| - \frac{\eta}{1 + 2\hat{\mu}\Delta t} $$
where $d$ is dimension, $\eta$ is viscosity, and $\Delta t$ is time step. The return mapping is:
$$ Z(\mathbf{F}) = \begin{cases} \mathbf{F} & \delta\gamma \le 0 \\ \mathbf{U} \exp\left(\frac{1}{2\mu}\hat{\mathbf{\epsilon}} + \frac{1}{d}\text{Tr}(\mathbf{\epsilon})\mathbf{I}\right)\mathbf{V}^T & \text{otherwise} \end{cases} $$
*Note: The source text for the viscoplastic return mapping formula is garbled (`U exp ( 1 2µ ˆ + d Tr(ε)1)V`). The formula above is interpreted as a standard viscoplastic return mapping structure based on context, but strictly quoting the text is not possible due to ambiguity [S1, unofficial].*

### Parameters
*   $\mu, \lambda$: Lamé parameters (typical values depend on material, e.g., rubber, plasticine).
*   $\tau_Y$: Yield stress for plasticine model.
*   $\eta$: Viscosity for non-Newtonian fluid.
*   $\Delta t$: Time step.

### Edge Cases and Pitfalls
*   **Precision**: Numerical instability may occur if $\Delta t$ is too large, especially for stiff materials or high viscosity [S1, unofficial].
*   **Boundaries**: Handling boundary conditions (e.g., contact with static objects) requires careful implementation of force computation and particle-grid interactions [S1, unofficial].
*   **Degenerate Input**: Singular deformation gradients (e.g., $J \le 0$) must be handled to prevent numerical errors [S1, unofficial].

## Subtopics

### [Neo-Hookean Elasticity](21557-neo-hookean-elasticity.md)
Neo-Hookean elasticity is a hyperelastic constitutive model used to simulate the large-deformation behavior of incompressible or nearly incompressible materials, such as soft tissues and rubber. It defines the Cauchy stress tensor $\mathbf{T}$ as a function of the deformation gradient $\mathbf{F}$ and the material's Lamé parameters. In the context of Gaussian Splatting and implicit neural representations, it serves as the physical prior for soft-body control, allowing shape changes to be computed via physical simulation rather than purely geometric deformation. It is distinguished from linear elasticity by its ability to handle finite strains and from plasticity models by its reversible, energy-conserving nature.

### StVK + von-Mises Plasticity Model
Simulates plasticine using St.Venant-Kirchhoff (StVK) constitutive model combined with von-Mises plastic return mapping. The stress associated with the model is defined by $\mathbf{T}(\mathbf{F}) = \mathbf{U}(2\mu\mathbf{\epsilon} + \lambda \text{Tr}(\mathbf{\epsilon}))\mathbf{U}^T$, where $\mathbf{F} = \mathbf{U}\mathbf{\Sigma}\mathbf{V}$ is the SVD of deformation gradient $\mathbf{F}$, and $\mathbf{\epsilon} = \log(\mathbf{\Sigma})$ is the Hencky strain. The von-Mises yielding condition is $\delta\gamma = \|\hat{\mathbf{\epsilon}}\| - \frac{\tau_Y}{2\mu} > 0$, where $\hat{\mathbf{\epsilon}}$ is the normalized Hencky strain. $\tau_Y$ is the yield stress, which is the parameter for this model. $\delta\gamma > 0$ means the deformation gradient is outside the elastic region. The deformation gradient will be projected back onto the boundary of elastic region (yield surface) by the return mapping $Z(\mathbf{F})$ [S1, unofficial].

### Viscoplastic Model
Models non-Newtonian fluid with yield stress using a viscoplastic model (Yue et al., 2015) with von-Mises criteria for the elastic region. Define $\hat{\mu} = \frac{\text{Tr}(\mathbf{\Sigma}^2)}{d}$, $\mathbf{s} = 2\hat{\mu}\hat{\mathbf{\epsilon}}$, $\hat{s} = \|\mathbf{s}\| - \frac{\eta}{1 + 2\hat{\mu}\Delta t}$, where $d$ is dimension, $\eta$ is viscosity, $\Delta t$ is time step. The return mapping $Z(\mathbf{F})$ is applied if $\delta\gamma > 0$. Note: The source text formula for the return mapping is garbled, so the exact implementation details are ambiguous [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
The Material Point Method (MPM) is a continuum mechanics framework that supports various constitutive and plasticity models [S1, unofficial]. It is used in the PhysGaussian method to provide a unified representation for both rendering and simulation of complex materials [S1, unofficial]. The specific implementations of Neo-Hookean, StVK, and Viscoplastic models are adapted from continuum mechanics literature and applied to the particle-based representation in PhysGaussian [S1, unofficial].

## Sources
[S1] PAC-NeRF: Physics Augmented Continuum Neural Radiance Fields for Geometry-Agnostic System Identification, https://arxiv.org/abs/2303.05512v1, fetched 2026-10-26 (unofficial)
