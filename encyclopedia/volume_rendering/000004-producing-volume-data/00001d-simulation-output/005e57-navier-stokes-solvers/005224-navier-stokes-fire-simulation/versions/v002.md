# Navier-Stokes Fire Simulation
Also called: Full 3D Finite Difference Solution, Physical Simulation (Navier-Stokes/Lattice), Physics-Based Fire, Physics-Based Fire Simulation

## Overview
Navier-Stokes Fire Simulation models fire as a hot, turbulent gas by solving the physical laws of fluid dynamics, specifically the Navier-Stokes equations, to generate time-varying volumetric data [S1, book][S2, unofficial]. It differs from procedural or particle-based siblings by deriving motion from conservation of momentum and mass rather than noise functions or ad-hoc particle rules, providing physically accurate convection and buoyancy [S1, book][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D spatial domain (grid) with resolution $(N_x, N_y, N_z)$, physical properties (viscosity, thermal expansion coefficient), and initial conditions (velocity field $\mathbf{u}$, temperature/density field) [G].
*   **Output:** A time-evolving velocity field $\mathbf{u}(x,y,z,t)$ and scalar field (density/temperature) suitable for volume rendering [G].

**Data Structures:**
*   **Grid:** A 3D array storing velocity vectors $\mathbf{u}$ and scalars (density/temperature) at cell centers or staggered locations [G].
*   **Resolution:** Flow calculations are performed on relatively coarse grids for efficiency [S1, book].

**Steps:**
1.  **Initialize:** Set initial velocity $\mathbf{u}_0$ and temperature/density fields [G].
2.  **Force Application:** Calculate external forces acting on the gas, primarily thermal buoyancy (upward force proportional to temperature difference) and drag [S2, unofficial].
3.  **Advection:** Transport velocity and scalar fields by the current velocity field, representing convection via Newton’s Laws of Motion [S2, unofficial].
4.  **Diffusion:** Apply viscous diffusion to the velocity field and thermal diffusion to the temperature field [S3, unofficial].
5.  **Pressure Projection:** Solve the incompressibility constraint ($\nabla \cdot \mathbf{u} = 0$) to compute a pressure field that corrects the velocity field to be divergence-free [G].
6.  **Render:** Use the resulting coarse volumetric data with global illumination algorithms (e.g., photon maps, Monte Carlo volume renderers) or splats texture mapped with phenomena detail to visually enhance the level of detail [S1, book].

**Formulas:**
The method solves simplified Navier-Stokes equations for incompressible gas [S3, unofficial]. The velocity evolution is:
$$ \frac{\partial \mathbf{u}}{\partial t} = v\nabla \cdot (\nabla \mathbf{u}) - (\mathbf{u} \cdot \nabla)\mathbf{u} - \nabla p + \mathbf{F}_{bv} $$
where $v$ is viscosity, $\nabla$ is the gradient operator, $\cdot$ is the inner product, $p$ is pressure, and $\mathbf{F}_{bv}$ is the buoyant force [S3, unofficial]. The buoyant force is defined as:
$$ \mathbf{F}_{bv} = -\beta g_v (T_0 - T_k) $$
where $\beta$ is the thermal expansion coefficient, $g_v$ is vertical gravity, $T_0$ is a reference temperature, and $T_k$ is the average temperature on the boundary between a gaseous cell and the one above it [S3, unofficial]. Temperature changes over time according to:
$$ \frac{\partial T}{\partial t} = \lambda \nabla \cdot (\nabla T) - \nabla \cdot (T \mathbf{u}) $$
where $\lambda$ represents the diffusion process, $T$ is temperature, and $\mathbf{u}$ is velocity [S3, unofficial].

**Parameters:**
*   **Grid Resolution:** Low resolution is standard for real-time or efficient simulation [S1, book][S2, unofficial].
*   **Viscosity ($v$) and Diffusion ($\lambda$):** Control flow smoothness and thermal dissipation rates [S3, unofficial].
*   **Buoyancy Coefficient ($\beta$):** Controls the strength of the upward thermal force [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Stability:** The equations are only stable for a certain period of time and require low resolution for efficiency [S3, unofficial].
*   **Detail Loss:** Coarse grids lack high-frequency detail, necessitating global illumination or detail-enhancing techniques for visual fidelity [S1, book].
*   **Numerical Precision:** Stability depends on appropriate time-stepping relative to grid resolution [G].

## Subtopics

### Foster and Metaxas Smoke/Gas Model
A physics-based smoke animation method using simplified Navier-Stokes equations for incompressible gas, influenced by vorticity, thermal buoyancy, turbulence, convection, and drag [S3, unofficial]. It models gas motion in a 3D cubic grid aligned to an x, y, z coordinate system [S3, unofficial]. The method produces realistic churning and floating smoke but is only stable for a limited duration and forces low resolution for efficiency [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** High. Modeling actual underlying physical processes via the Navier-Stokes equations requires much larger computational effort compared to procedural methods [S1, book].
*   **Memory Complexity:** Proportional to the grid volume $(N_x \cdot N_y \cdot N_z)$, storing velocity and scalar fields [G].
*   **Performance:** Historically slow, but recent advances in graphics hardware have yielded powerful SIMD processors that can run the required numerical solvers at speedups of an order of magnitude or more compared to traditional CPUs [S1, book].
*   **Strengths:** Physically accurate behavior, realistic convection and buoyancy, natural turbulence [S1, book][S2, unofficial].
*   **Limitations:** Computationally intensive, requires post-processing (global illumination) to achieve high visual detail due to coarse grid usage [S1, book].
*   **When to Use:** When physical accuracy is prioritized over raw performance or procedural control [G].
*   **When Not to Use:** When real-time performance on low-end hardware is critical without hardware acceleration, or when artistic control via noise functions is preferred [G].

## Background
Originating from the application of fluid dynamics laws to computer graphics, this method was developed to improve upon procedural noise-based fire rendering [S1, book][S2, unofficial]. It represents fire as hot and turbulent gas, solving volumetric differential equations at low resolutions [S2, unofficial]. The approach leverages advances in graphics hardware (SIMD processors) to make the numerical solvers feasible [S1, book].

## Sources
[S1] PDF Volume Visualization and Volume Graphics, https://www3.cs.stonybrook.edu/~mueller/teaching/cse332/volvisChapter.pdf, fetched 2026-10-28 (book)
[S2] Realistic Real-Time Rendering of Fire in a Production System, https://clas.iusb.edu/math-compsci/_prior-proposals/YVanzine_Proposal.pdf, fetched 2026-10-28 (unofficial)
[S3] Efficient computation and rendering of smoke animation, http://hdl.handle.net/2346/16985, fetched 2026-10-26 (unofficial)
