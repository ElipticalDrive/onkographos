# Lennard-James Potential Cloud Simulation
Also called: Elhaddad Clouds
## Overview
Simulates clouds using particles by approximating the force between these particles based on the Lennard-Jones potential [S1]. It differs from Position Based Fluids [1765-position-based-fluids.md] by using a specific inter-particle potential function rather than constraint-based position correction, and from Massless Particle Systems [3970-massless-particle-system.md] by explicitly modeling repulsive and attractive forces to maintain cloud structure [S1].
## How it works
**Inputs and Outputs:**
*   **Input:** A set of cloud particles, each with position, velocity, and potentially scalar fields (e.g., temperature, density).
*   **Output:** Updated particle positions and velocities for the next simulation frame.

**Data Structures:**
*   **Particle:** A structure containing:
    *   `position`: 3D vector $(x, y, z)$.
    *   `velocity`: 3D vector $(v_x, v_y, v_z)$.
    *   `mass`: Scalar value $m$.
    *   `radius`: Interaction radius $r_c$.
    *   `scalar_fields`: Optional map of scalar values (e.g., temperature).

**Method:**
1.  **Neighbor Search:** For each particle $i$, identify neighboring particles $j$ within a cutoff distance $r_c$.
2.  **Force Calculation:** Compute the force $\mathbf{F}_{ij}$ exerted by neighbor $j$ on particle $i$ using the Lennard-Jones potential derivative. The force is repulsive at short distances and attractive at longer distances within the cutoff.
    *   Let $r_{ij} = \|\mathbf{r}_j - \mathbf{r}_i\|$ be the distance between particles.
    *   The potential energy $U(r_{ij})$ is typically defined as:
        $$ U(r_{ij}) = 4\epsilon \left[ \left(\frac{\sigma}{r_{ij}}\right)^{12} - \left(\frac{\sigma}{r_{ij}}\right)^{6} \right] $$
        where $\epsilon$ is the depth of the potential well and $\sigma$ is the finite distance at which the inter-particle potential is zero [S1].
    *   The force is the negative gradient:
        $$ \mathbf{F}_{ij} = -\nabla U(r_{ij}) = 24\epsilon \left[ 2\left(\frac{\sigma}{r_{ij}}\right)^{12} - \left(\frac{\sigma}{r_{ij}}\right)^{6} \right] \frac{\mathbf{r}_{ij}}{r_{ij}^2} $$
        where $\mathbf{r}_{ij} = \mathbf{r}_j - \mathbf{r}_i$.
    *   [G] In practice, the force is often truncated at $r_c = 2.5\sigma$ to zero to save computation.
3.  **Integration:** Update velocity and position using a standard integrator (e.g., Velocity Verlet or Euler).
    *   $\mathbf{v}_i(t + \Delta t) = \mathbf{v}_i(t) + \frac{\Delta t}{m_i} \sum_{j \neq i} \mathbf{F}_{ij}$
    *   $\mathbf{r}_i(t + \Delta t) = \mathbf{r}_i(t) + \Delta t \cdot \mathbf{v}_i(t + \Delta t)$
4.  **Boundary Handling:** Apply boundary conditions to keep particles within the simulation domain.

**Parameters:**
*   $\epsilon$: Potential well depth. Typical value: Not given in the sources.
*   $\sigma$: Particle diameter. Typical value: Not given in the sources.
*   $r_c$: Cutoff radius. Typical value: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Singularity:** If $r_{ij} \to 0$, the repulsive term $(\sigma/r_{ij})^{12}$ explodes. A minimum distance clamp or softening factor is required [G].
*   **Cutoff Artifacts:** Discontinuities in force at $r_c$ can cause energy drift. A smoothing function is often applied near $r_c$ [G].
*   **Precision:** Floating-point precision issues may arise in neighbor search for large-scale simulations.

## Subtopics

### GPU Particle-Based Cloud Simulation (Barbosa)
A particle-based cloud simulation on the GPU where particle physics is handled using position-based dynamics [S1]. Cloud dynamics leverages principles from the referenced section 4.2.1 [S1]. Cloud detail is achieved by using adaptive particles, which can be merged or split in regions where clouds disappear or more details emerge, respectively [S1]. Additionally, scalar fields such as temperature are smoothened out to reduce the value difference between neighboring particles [S1]. This variant differs from the standard Lennard-Jones approach by using position-based dynamics for stability and adaptive resolution for detail [S1].

## Operating characteristics
**Time and Memory Complexity:**
*   Time complexity: $O(N^2)$ for naive neighbor search, $O(N)$ with spatial hashing or grid-based neighbor search.
*   Memory complexity: $O(N)$ for storing particle data.

**Measured Performance:**
*   Not given in the sources.

**Scaling:**
*   Scales with the number of particles $N$. GPU implementation allows for parallel processing of particle interactions [S1].

**Strengths and Limitations:**
*   **Strengths:** Physically plausible cloud shapes; handles complex interactions like vortex formation [S1].
*   **Limitations:** Computationally expensive for large $N$; requires careful tuning of $\epsilon$ and $\sigma$ [S1].

**When to Use:**
*   When physically accurate cloud dynamics are required and computational resources allow for particle-based simulation.

**When Not to Use:**
*   When real-time performance on low-end hardware is critical without GPU acceleration.

## Background
Originated from Elhaddad et al. [S1]. The method simulates clouds using particles by approximating the force between these particles based on the Lennard-Jones potential [S1]. It is part of the broader family of particle-based cloud simulations, which also includes position-based dynamics approaches like Barbosa et al. [S1].

## Sources
[S1] A survey of modeling, rendering and animation of clouds in computer graphics, https://doi.org/10.1007/s00371-020-01953-y, fetched 2026-10-26
