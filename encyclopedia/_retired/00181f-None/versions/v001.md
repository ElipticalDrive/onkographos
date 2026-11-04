# Stochastic Volume Rendering of Multi-Phase SPH Data

Also called: Coalesced Memory Access Sampling, GPU Ray Marching Implementation, OpenCL SPH Renderer

## Overview
A direct volume rendering technique for large, unstructured smoothed-particle hydrodynamics (SPH) data that avoids intermediate grid representations by stochastically sampling a subset of particles during ray marching [S1]. It differs from standard DVR by using view-dependent and spatial-complexity-based probabilities to select particles, enabling continuous scaling from interactive previews to high-quality renders [S1]. It specifically addresses multi-phase flows by reconstructing interfaces between different fluid types (e.g., gas and liquid) using a combined volumetric and surface shading model [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   SPH particle dataset: Each particle $i$ has position $\mathbf{p}_i$, mass $m_i$, density $\rho_i$, phase ID, and other attributes (e.g., temperature, velocity).
*   Camera parameters: Eye position, view direction, projection matrix.
*   Rendering parameters: Number of ray march steps, kernel type (cubic spline or Epanechnikov), maximum number of particles to sample per cell, random seed/table.
*   Transfer functions: For volumetric opacity and color.
*   Light sources: For shading.

**Outputs:**
*   Rendered image: Color and opacity per pixel.

### Data Structures

**1. Particle Storage**
*   Flat array of particles.
*   Particles are grouped by the grid cell they reside in.
*   Each particle $i$ stores:
    *   $\mathbf{p}_i \in \mathbb{R}^3$: Position.
    *   $m_i$: Mass.
    *   $\rho_i$: Density.
    *   $\phi_i$: Phase ID (integer).
    *   Other attributes (e.g., temperature $T_i$).

**2. Acceleration Structure (3D Uniform Grid)**
*   A 3D uniform grid covering the simulation domain.
*   Each cell stores a reference (pointer/index) to a contiguous block of memory in the Particle Storage array containing all particles within that cell.
*   This structure allows $O(1)$ access to the list of particles in a specific spatial region [S1].
*   Note: The grid is used *only* for querying particles, not as a discrete signal for traversal [S1].

### Numbered Steps

1.  **Ray Initialization**: For each pixel, generate a primary ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$.
2.  **Ray Marching Loop**: Iterate along the ray for a fixed number of steps or until the ray exits the domain.
    a.  **Determine Sampling Position**: Calculate the current sampling point $\mathbf{x}$ on the ray.
    b.  **Identify Relevant Grid Cell**: Find the grid cell containing $\mathbf{x}$.
    c.  **Stochastic Particle Sampling**:
        i.  Retrieve the list of particles in the current cell from the acceleration structure.
        ii. Select a subset of these particles using stochastic sampling. The probability of selecting a particle is determined by an importance measure based on view-dependent factors and spatial complexity [S1].
        iii. To ensure coalesced memory access on GPUs, all threads in a work group share the same random offset, but their local work group index is added as an offset to the selection [S1]. The selected subset is always a contiguous block of particles in memory [S1].
    d.  **SPH Evaluation**:
        i.  For each selected particle $j$, compute the kernel weight $W(\mathbf{x} - \mathbf{p}_j)$ using the chosen kernel function (cubic spline or Epanechnikov) [S1].
        ii. Estimate the field value (e.g., density, phase indicator) at $\mathbf{x}$ by summing contributions from selected particles:
            $$ V(\mathbf{x}) = \sum_{j \in \text{subset}} m_j \frac{W(\mathbf{x} - \mathbf{p}_j)}{\rho_j} $$
        iii. Determine the dominant phase at $\mathbf{x}$ based on the phase IDs of the contributing particles.
        iv. Compute gradients if needed for surface reconstruction [S1].
    e.  **Shading**:
        i.  **Volumetric Shading**: If the ray segment is within a homogeneous phase, use pre-integrated transfer functions [EKE01] for opacity and color accumulation. Note: Pre-integration assumes fixed segment length and cannot handle phase transitions [S1].
        ii. **Surface Shading**: If the ray segment crosses a phase interface, use a multi-material model. Apply Blinn-Phong BRDF for surface shading [S1].
        iii. **Single Scattering**: Optionally perform stochastic single scattering to improve spatial perception of phase interfaces [S1].
    f.  **Accumulation**: Accumulate color and opacity along the ray using standard volume rendering integral equations.
3.  **Final Color**: Output the accumulated color for the pixel.

### Formulas

**SPH Kernel Functions:**
*   **Epanechnikov Kernel** (support radius $h$):
    $$ W(r) = \frac{3}{4\pi h^3} \left(1 - \frac{r^2}{h^2}\right) \quad \text{for } r \le h $$
*   **Cubic Spline Kernel** (support radius $2h$):
    $$ W(r) = \frac{10}{7\pi h^3} \begin{cases} 1 - \frac{3}{2}q^2 + \frac{3}{4}q^3 & 0 \le q < 1 \\ \frac{1}{4}(2-q)^3 & 1 \le q < 2 \\ 0 & q \ge 2 \end{cases} $$
    where $q = r/h$. The cubic spline is smoother but has twice the support width, requiring more particle accesses [S1].

**Importance Sampling Probability:**
*   The probability $P_i$ of selecting particle $i$ is proportional to an importance measure $I_i$. The exact formula for $I_i$ is not provided in the sources, but it is derived from view-dependent factors and spatial complexity [S1].

**Pre-integrated Transfer Function:**
*   For homogeneous segments, the transmittance $T$ and color $C$ are pre-computed in a 2D texture based on segment length $L$ and optical depth $\tau = \int \sigma_t ds$ [EKE01]. This is not applicable to segments with phase transitions [S1].

### Parameters

*   **Kernel Function**: User-specified. Options: Cubic spline (smoother, wider support, more accesses) or Epanechnikov [S1].
*   **Shading Model**: Blinn-Phong BRDF for surface, Henyey-Greenstein phase function for volume [S1].
*   **Number of Particles to Sample**: Controls quality vs. performance. Higher values yield more accurate results [S1].
*   **Random Values**: Precomputed pseudo-random values are used instead of generating them on the GPU [S1].

### Edge Cases and Pitfalls

*   **Random Number Generation**: Generating pseudo-random numbers on GPUs is difficult and slow. The method uses precomputed random values to avoid this [S1].
*   **Phase Transitions**: Pre-integrated transfer functions cannot be used for ray segments that cross phase boundaries because they assume fixed segment length and homogeneous material [S1].
*   **Grid Usage**: The uniform grid is only for particle querying. Digital Differential Analyzer (DDA) or higher-order traversal methods are not applicable because the grid does not represent a discrete signal [S1].
*   **Memory Coalescing**: To ensure efficient GPU memory access, the selected subset of particles must be contiguous in memory. This is achieved by sharing a random offset across a work group and adding the local work group index [S1].

## Operating characteristics

*   **Time Complexity**: Depends on the number of particles sampled per step. By sampling a subset, it reduces the complexity from $O(N)$ per step to $O(k)$ where $k \ll N$. The overall complexity is $O(R \cdot S \cdot k)$ where $R$ is the number of rays, $S$ is the number of steps per ray, and $k$ is the number of sampled particles per step [S1].
*   **Memory Complexity**: $O(N)$ for storing particles and the grid. The grid adds minimal overhead as it only stores references [S1].
*   **Performance**: Scales continuously from fast, interactive previews to high-quality rendering by adjusting the number of sampled particles [S1].
*   **Strengths**: Minimal preprocessing, no intermediate grid representation, handles large datasets, supports multi-phase flows, efficient GPU implementation with coalesced memory access [S1].
*   **Limitations**: Stochastic sampling introduces noise; requires careful tuning of the importance measure and number of samples for quality [S1]. Pre-integration is limited to homogeneous segments [S1].
*   **When to Use**: Large SPH datasets where preprocessing time and memory are constraints, and interactive or semi-interactive visualization is desired. Particularly suitable for multi-phase flows [S1].
*   **When Not to Use**: When exact, noise-free rendering is required without post-processing, or when the dataset is small enough that full particle evaluation is feasible [S1].

## Background

*   **Origin**: The method was introduced in the paper "Stochastic Volume Rendering of Multi-Phase SPH Data" [S1].
*   **History**: It builds on previous work in SPH visualization and stochastic rendering. It addresses the computational complexity of rendering large, unstructured particle data by avoiding intermediate representations and using importance sampling [S1].
*   **Usage**: Used in scientific visualization for fluid dynamics, particularly for multi-phase flows like gas-liquid interactions [S1].

## Sources

[S1] Stochastic Volume Rendering of Multi‐Phase SPH Data, https://doi.org/10.1111/cgf.14121, fetched 2026-10-27
