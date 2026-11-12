# PhysGaussian
Also called: MPM over Gaussians
## Overview
PhysGaussian couples 3D Gaussian Splatting (3DGS) with continuum mechanics via the Material Point Method (MPM) to enable simultaneous high-fidelity rendering and physical simulation of deformable and granular materials [S1, S2, unofficial]. It distinguishes itself from standard 3DGS by embedding physical properties into each Gaussian primitive, allowing the representation to evolve under dynamic forces such as deformation, fracture, and fluid flow [S1, S2, unofficial].
## How it works
PhysGaussian extends the static Gaussian model by associating physical state variables with each Gaussian particle, enabling the use of MPM for simulation while retaining the differentiable rasterization pipeline for rendering [S1, S2, unofficial].

**Inputs and Outputs:**
*   **Input:** A static 3DGS scene (positions, covariances, colors, opacities) representing a deformable object or environment.
*   **Output:** A time-evolving sequence of rendered frames and physical state updates (displacements, velocities, stresses) consistent with the simulated material behavior [S1, S2, unofficial].

**Data Structures:**
Each Gaussian $i$ in the scene maintains standard 3DGS attributes plus physical fields [S1, S2, unofficial]:
1.  **Position:** $\mathbf{x}_i \in \mathbb{R}^3$ (center of the Gaussian).
2.  **Covariance:** $\Sigma_i \in \mathbb{R}^{3 \times 3}$ (shape/orientation).
3.  **Color:** $\mathbf{c}_i$ (appearance).
4.  **Opacity:** $\alpha_i$.
5.  **Physical State:**
    *   **Mass:** $m_i$ (derived from density and volume).
    *   **Velocity:** $\mathbf{v}_i \in \mathbb{R}^3$.
    *   **Deformation Gradient:** $\mathbf{F}_i \in \mathbb{R}^{3 \times 3}$ (local deformation).
    *   **Stress:** $\boldsymbol{\sigma}_i \in \mathbb{R}^{3 \times 3}$ (internal forces).

**Method Steps:**
1.  **Initialization:** Initialize Gaussians from the input 3DGS reconstruction. Assign initial physical properties (density, material parameters) based on the visual appearance or user specification [S1, S2, unofficial].
2.  **MPM Simulation Step:**
    *   Map Gaussian properties to a background grid (mass, momentum, force) using kernel interpolation [S1, S2, unofficial].
    *   Update grid velocities and positions using the equations of motion derived from the constitutive model (e.g., Neo-Hookean for cloth, Drucker-Prager for sand) [S1, S2, unofficial].
    *   Map grid updates back to Gaussians to update $\mathbf{x}_i$, $\mathbf{v}_i$, and $\mathbf{F}_i$ [S1, S2, unofficial].
3.  **Rendering Step:**
    *   Update Gaussian covariances $\Sigma_i$ based on the new deformation gradient $\mathbf{F}_i$ and orientation [S1, S2, unofficial].
    *   Rasterize the updated Gaussians using the standard differentiable 3DGS splatting algorithm to produce the image [S1, S2, unofficial].
4.  **Differentiable Loop (Optional):** If training or editing, backpropagate through the simulation step (if differentiable) or the rendering step to optimize physical parameters or initial states [S1, S2, unofficial].

**Parameters:**
*   **Material Model:** Neo-Hookean, Drucker-Prager, etc. [S1, S2, unofficial].
*   **Grid Resolution:** MPM background grid spacing. [Not given in the sources.]
*   **Time Step:** $\Delta t$ for simulation. [Not given in the sources.]

**Edge Cases and Pitfalls:**
*   **Tearing/Fracture:** Requires specific constitutive models or damage criteria to handle fracture correctly [S1, unofficial].
*   **Granular Flow:** Requires appropriate friction and pressure models for materials like sand or snow [S2, unofficial].
*   **Precision:** Coupling discrete Gaussians with continuous grid-based MPM requires careful interpolation to avoid energy loss or instability [S1, S2, unofficial].
## Operating characteristics
*   **Time Complexity:** Dominated by the MPM simulation step (grid operations) and the 3DGS rasterization step. Scaling depends on the number of Gaussians and grid resolution. [Not given in the sources.]
*   **Memory Complexity:** Higher than standard 3DGS due to additional physical state variables per Gaussian. [Not given in the sources.]
*   **Strengths:** Unified representation for rendering and simulation; supports diverse materials (cloth, jelly, sand, snow); enables interactive physics-based editing. [S1, S2, unofficial]
*   **Limitations:** Simulation complexity; requires careful tuning of physical parameters; potential instability in extreme deformations. [S1, S2, unofficial]
*   **Use Cases:** Virtual reality, robotics simulation, visual effects, and interactive editing of dynamic scenes. [S2, unofficial]
## Background
PhysGaussian was introduced at CVPR 2024 [S2, unofficial]. It represents a convergence of neural rendering and computational physics, leveraging the efficiency of 3D Gaussian Splatting for visualization and the robustness of the Material Point Method for physical simulation [S1, S2, unofficial]. It is particularly useful for applications requiring both visual fidelity and physical plausibility, such as autonomous vehicle simulation or robotics training [S2, unofficial].
## Sources
[S1] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
