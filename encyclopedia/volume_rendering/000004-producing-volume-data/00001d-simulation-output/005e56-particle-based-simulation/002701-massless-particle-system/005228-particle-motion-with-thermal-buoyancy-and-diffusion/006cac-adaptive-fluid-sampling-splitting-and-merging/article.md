# Adaptive Fluid Sampling (Splitting and Merging)
Also called: Adaptive SPH Sampling, Dynamic Particle Subdivision and Aggregation

## Overview
Adaptive Fluid Sampling is a particle management technique for Smoothed Particle Hydrodynamics (SPH) that dynamically adjusts particle density to balance visual fidelity and computational cost. It solves the problem of inefficient uniform sampling by splitting particles in geometrically complex surface regions to improve accuracy and merging particles in dense, deep-fluid regions to reduce the particle count. It differs from static sampling by using local neighborhood geometry to drive subdivision and aggregation, and from purely visual decimation by maintaining physical consistency through mass-weighted attribute averaging during merging [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of SPH particles, each with position $\mathbf{x}_i$, mass $m_i$, density $\rho_i$, velocity $\mathbf{v}_i$, and smoothing length $h_i$. A neighbor search structure (e.g., bucket grid) is required to identify neighborhoods.
*   **Output:** A modified set of particles where surface particles have been subdivided into sub-particles and inner particles have been aggregated into merged particles.

**Data Structures:**
*   **Particle Buffer:** Stores attributes for all active particles.
*   **Bucket Space:** A 3D grid of voxel cells with grid size $s$. Each particle is assigned to the closest voxel cell to facilitate neighbor search and ensure spatial locality for merging [S1].

**Parameters:**
*   $\alpha$: User-defined constant determining the distance from the parent particle to its sub-particles. Typical values depend on the desired resolution of surface detail.
*   $\delta$: User-defined parameter to handle extreme cases (e.g., splashes) where symmetric neighbor distribution masks surface status.
*   $s$: Grid size of the bucket space.
*   $\sigma$: Constant related to the smoothing kernel support (implied by Equation 3 in [S1], typically related to the smoothing length definition).
*   $\rho$: Fluid density.

**Steps:**

1.  **Compute Center of Mass for Neighborhoods:**
    For each particle $p$, identify its neighborhood particles $i$. Compute the corrected mass $m_{c_i}$ for each neighbor (as defined in the parent SPH method [S1]). Calculate the center of mass $\chi_p$ of the neighborhood:
    $$ \chi_p = \frac{\sum m_{c_i} \mathbf{x}_i}{\sum m_{c_i}} $$
    where $\mathbf{x}_i$ is the position of particle $i$ in $p$'s neighborhood [S1].

2.  **Identify Surface Particles:**
    A particle $p$ is marked as a surface particle if its distance to the center of mass $\chi_p$ exceeds a certain threshold. To handle extreme cases where neighbors distribute symmetrically (making $p$ appear close to $\chi_p$ despite being on the surface), add the criterion:
    $$ \sum m_{c_i} m_p < \delta $$
    If this condition is met, $p$ is also marked as a surface particle [S1].

3.  **Identify Splittable Particles:**
    A surface particle is splittable if its mass $m_p$ is greater than the minimum mass threshold $\varepsilon_m$. The threshold $\varepsilon_m$ is derived to prevent sub-particles from mapping to the same voxel in the bucket space:
    $$ \varepsilon_m \geq \rho \left( \frac{\sqrt{9/8} s}{\alpha \sigma} \right)^3 $$
    (Note: The source formula [S1] is written as $\varepsilon_m \geq \rho \left( \frac{\sqrt{9/8s}}{\alpha \sigma} \right)^3$, but dimensional analysis and the derivation step $\sqrt{\frac{8}{3}} \alpha \sigma (\frac{\varepsilon_m}{\rho})^{1/3} \geq \sqrt{3}s$ imply the $s$ is in the numerator inside the cube or the term is $\frac{\sqrt{9/8} s}{\alpha \sigma}$. We follow the explicit inequality derivation: $ds = \sqrt{8/3} \alpha h_p \geq \sqrt{3} s$. Using $h_p = \sigma (\frac{m_p}{\rho})^{1/3}$, we get $\sqrt{8/3} \alpha \sigma (\frac{\varepsilon_m}{\rho})^{1/3} \geq \sqrt{3} s \Rightarrow (\frac{\varepsilon_m}{\rho})^{1/3} \geq \frac{\sqrt{3} s}{\sqrt{8/3} \alpha \sigma} = \frac{3 s}{\sqrt{8} \alpha \sigma}$. Cubing both sides: $\frac{\varepsilon_m}{\rho} \geq \frac{27 s^3}{8 \alpha^3 \sigma^3}$. Thus $\varepsilon_m \geq \rho \frac{27}{8} (\frac{s}{\alpha \sigma})^3$. The source text has a typo in the final formula representation or variable definitions. We cite the source's explicit formula but note the derivation logic [S1].)
    *Correction based on strict source citation:* The source states: $\varepsilon_m \geq \rho \left( \frac{\sqrt{9/8s}}{\alpha \sigma} \right)^3$. We use this exact form as provided in the source, acknowledging potential transcription ambiguity in the original paper's typesetting [S1].

4.  **Splitting Operation:**
    For each splittable particle $p$:
    a.  **Local Coordinate System:** Construct a local frame centered at $\mathbf{x}_p$:
        *   $\mathbf{Z} = \text{normalize}(\mathbf{x}_p - \chi_p)$
        *   $\mathbf{X} = \text{normalize}(-Z_z, 0, Z_x)$
        *   $\mathbf{Y} = \text{cross}(\mathbf{X}, \mathbf{Z})$
    b.  **Sub-particle Positions:** Generate four sub-particles $p_i$ ($i=1..4$) at the corners of a tetrahedron centered at $\mathbf{x}_p$. The distance $d$ from $\mathbf{x}_p$ to each $p_i$ is:
        $$ d = \alpha h_p $$
        where $h_p$ is the smoothing length of particle $p$. The positions are $\mathbf{x}_{p_i} = \mathbf{x}_p + d \cdot \mathbf{v}_i$, where $\mathbf{v}_i$ are the four tetrahedral directions in the local frame [S1].
    c.  **Sub-particle Attributes:**
        *   Mass: $m_{p_i} = m_p / 4$
        *   Density: $\rho_{p_i} = \rho_p$
        *   Velocity: $\mathbf{v}_{p_i} = \mathbf{v}_p$
        *   Smoothing Length: $h_{p_i}$ is computed using Equation 3 from the source (typically $h \propto m^{1/3}$) [S1].
    d.  **Replacement:** Remove $p$ and add the four sub-particles to the particle buffer.

5.  **Identify Inner Particles:**
    A particle $p$ is marked as an inner particle if it is **not** a surface particle and its distance to the center of mass $\chi_p$ is below a certain threshold (indicating it is deep inside the fluid, surrounded by neighbors) [S1].

6.  **Merging Operation:**
    Group nearby inner particles $p_i$ into clusters. For each cluster, create a single aggregate particle $p_{merged}$:
    a.  **Mass:** $m_{merged} = \sum m_i$
    b.  **Position:** $\mathbf{x}_{merged} = \frac{\sum m_i \mathbf{x}_i}{\sum m_i}$ (mass-weighted average)
    c.  **Density:** $\rho_{merged} = \frac{\sum m_i \rho_i}{\sum m_i}$ (mass-weighted average)
    d.  **Velocity:** $\mathbf{v}_{merged} = \frac{\sum m_i \mathbf{v}_i}{\sum m_i}$ (mass-weighted average)
    e.  **Smoothing Length:** $h_{merged}$ is computed using Equation 3 from the source [S1].
    f.  **Replacement:** Remove the original inner particles and add $p_{merged}$ [S1].

**Pitfalls:**
*   **Pressure Artifacts:** Merged particles have larger mass, resulting in increased pressure on neighboring particles. This repulsive force can prevent other particles from coming close enough to be merged, potentially stalling the merging process or causing instability [S1].
*   **GPU Implementation:** The splitting operation is straightforward in a geometry shader with stream output. The merging operation is more complex on the GPU due to the need for efficient neighbor finding and cluster identification without data dependencies [S1].

## Operating characteristics
*   **Time Complexity:** The splitting and merging operations are performed per particle based on local neighborhood queries. With an efficient spatial hash (bucket space), neighbor search is $O(1)$ on average. The overall complexity is linear with respect to the number of particles $N$, $O(N)$, assuming constant average neighborhood size [S1].
*   **Memory Complexity:** The number of particles fluctuates. Splitting increases memory usage temporarily, while merging decreases it. The bucket space requires $O(V)$ memory where $V$ is the number of voxels [S1].
*   **Performance:** The method is designed for GPU implementation to achieve interactive performance. By reducing particle count in dense regions, it significantly improves simulation speed compared to uniform high-resolution sampling [S1].
*   **Strengths:** Improves visualization quality at surfaces; reduces computational cost in deep fluid regions; fully parallelizable on GPU.
*   **Limitations:** Merged particles may cause pressure artifacts; requires careful tuning of $\alpha$, $\delta$, and $\varepsilon_m$; complex to implement merging efficiently on GPU due to dynamic data structure updates [S1].

## Background
Adaptive Fluid Sampling was introduced in the context of GPU-based SPH simulation to address the limitations of uniform particle distributions. The method builds on earlier work by [APKG07] but modifies it to be GPU-friendly, avoiding expensive operations that do not map well to parallel architectures. It was developed to enable interactive simulation and rendering of fluids on the GPU [S1].

## Sources
[S1] Adaptive Sampling and Rendering of Fluids on the GPU, https://doi.org/10.2312/vg/vg-pbg08/137-146, fetched 2026-10-26
