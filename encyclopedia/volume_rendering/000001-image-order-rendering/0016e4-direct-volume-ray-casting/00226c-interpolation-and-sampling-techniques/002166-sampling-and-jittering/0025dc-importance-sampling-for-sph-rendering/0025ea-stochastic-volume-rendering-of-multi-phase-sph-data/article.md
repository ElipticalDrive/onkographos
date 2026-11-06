# Stochastic Volume Rendering of Multi-Phase SPH Data

Also called: Coalesced Memory Access Sampling, GPU Ray Marching Implementation, OpenCL SPH Renderer

## Overview
A direct volume rendering technique for large, unstructured smoothed-particle hydrodynamics (SPH) data that avoids intermediate grid representations by stochastically sampling a subset of particles during ray marching [S1]. It differs from standard DVR by using view-dependent and spatial-complexity-based probabilities to select particles, enabling continuous scaling from interactive previews to high-quality renders [S1]. It specifically addresses multi-phase flows by reconstructing interfaces between different fluid types using a combined volumetric and surface shading model [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   SPH particle dataset containing positions, masses, densities, phase IDs, and attributes [S1].
*   Camera parameters: eye position, view direction, and projection matrix [G].
*   Rendering parameters: number of ray march steps, kernel type (cubic spline or Epanechnikov), maximum number of particles to sample per cell, and a precomputed random value table [S1].
*   Transfer functions for volumetric opacity and color [G].
*   Light sources for shading calculations [G].

**Outputs:**
*   Rendered image with color and opacity per pixel [G].

### Data Structures

**1. Particle Storage**
*   Flat array of particles grouped by the grid cell they reside in [S1].
*   Each particle $i$ stores:
    *   $\mathbf{p}_i \in \mathbb{R}^3$: Position [S1].
    *   $m_i$: Mass [S1].
    *   $\rho_i$: Density [S1].
    *   $\phi_i$: Phase ID (integer) [S1].
    *   Other attributes (e.g., temperature $T_i$) [G].

**2. Acceleration Structure (3D Uniform Grid)**
*   A 3D uniform grid covering the simulation domain [S1].
*   Each cell stores a reference (pointer/index) to a contiguous block of memory in the Particle Storage array containing all particles within that cell [S1].
*   This structure allows $O(1)$ access to the list of particles in a specific spatial region [S1].
*   The grid is used *only* for querying particles, not as a discrete signal for traversal [S1].

### Numbered Steps

1.  **Ray Initialization**: For each pixel, generate a primary ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ [G].
2.  **Ray Marching Loop**: Iterate along the ray for a fixed number of steps or until the ray exits the domain [G].
    a.  **Determine Sampling Position**: Calculate the current sampling point $\mathbf{x}$ on the ray [S1].
    b.  **Identify Relevant Grid Cell**: Find the grid cell containing $\mathbf{x}$ [S1].
    c.  **Stochastic Particle Sampling**:
        i.  Retrieve the list of particles in the current cell from the acceleration structure [S1].
        ii. Select a subset of these particles using stochastic sampling [S1]. The probability of selecting a particle is determined by an importance measure based on view-dependent factors and spatial complexity [S1].
        iii. To ensure coalesced memory access on GPUs, all threads in a work group share the same random offset, but their local work group index is added as an offset to the selection [S1]. The selected subset is always a contiguous block of particles in memory [S1].
    d.  **SPH Evaluation**:
        i.  For each selected particle $j$, compute the kernel weight $W(\mathbf{x} - \mathbf{p}_j)$ using the chosen kernel function (cubic spline or Epanechnikov) [S1].
        ii. Estimate the field value (e.g., density, phase indicator) at $\mathbf{x}$ by summing contributions from selected particles [S1].
        iii. Determine the dominant phase at $\mathbf{x}$ based on the phase IDs of the contributing particles [S1].
        iv. Compute gradients if needed for surface reconstruction [S1].
    e.  **Shading**:
        i.  **Volumetric Shading**: If the ray segment is within a homogeneous phase, use pre-integrated transfer functions [EKE01] for opacity and color accumulation [S1]. Pre-integration assumes fixed segment length and cannot handle phase transitions [S1].
        ii. **Surface Shading**: If the ray segment crosses a phase interface, use a multi-material model [S1]. Apply Blinn-Phong BRDF for surface shading [S1].
        iii. **Single Scattering**: Optionally perform stochastic single scattering to improve spatial perception of phase interfaces [S1].
    f.  **Accumulation**: Accumulate color and opacity along the ray using standard volume rendering integral equations [G].
3.  **Final Color**: Output the accumulated color for the pixel [G].

### Formulas

**SPH Kernel Functions:**
*   The exact mathematical expressions for the kernel functions are Not given in the sources.
*   The cubic spline kernel is smoother but has twice the support width of the Epanechnikov kernel, requiring more particle accesses [S1].

**Importance Sampling Probability:**
*   The probability $P_i$ of selecting particle $i$ is proportional to an importance measure $I_i$ [S1]. The exact formula for $I_i$ is Not given in the sources, but it is derived from view-dependent factors and spatial complexity [S1].

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

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_sph_volume(ray_origin, ray_dir, grid, particles, random_table, 
                      num_steps, kernel_type, max_samples_per_cell, 
                      workgroup_id, local_id, transfer_func):
    # Initialize accumulation variables [G]
    color = [0.0, 0.0, 0.0]
    alpha = 0.0
    t = 0.0
    
    # Ray marching loop [G]
    for step in range(num_steps):
        # 1. Determine sampling position [S1]
        x = ray_origin + t * ray_dir
        
        # 2. Identify relevant grid cell [S1]
        cell_idx = grid.query_cell(x)
        cell_particles = grid.cells[cell_idx].particle_block
        
        # 3. Stochastic particle sampling [S1]
        if len(cell_particles) == 0:
            continue
            
        # Use precomputed random table with work-group offset for coalesced access [S1]
        base_offset = random_table[workgroup_id]
        offset = base_offset + local_id
        sample_idx = offset % len(cell_particles)
        
        # Select contiguous block of particles [S1]
        k = min(max_samples_per_cell, len(cell_particles))
        subset = cell_particles[sample_idx : sample_idx + k]
        
        # 4. SPH evaluation [S1]
        density = 0.0
        phase_id = -1
        gradient = [0.0, 0.0, 0.0]
        
        for p in subset:
            r = distance(x, p.position)
            w = evaluate_kernel(r, p.h, kernel_type)
            density += p.mass * w / p.density
            if phase_id == -1 or p.phase_id == phase_id:
                phase_id = p.phase_id
            gradient += p.mass * w * (x - p.position) / p.density
            
        # 5. Shading [S1]
        # Check for phase transition or homogeneous segment [S1]
        if is_phase_transition(phase_id, previous_phase):
            # Surface shading with Blinn-Phong BRDF [S1]
            shading = compute_surface_shading(x, gradient, light_sources)
        else:
            # Volumetric shading with pre-integrated transfer functions [S1]
            optical_depth = compute_optical_depth(density, step_length)
            shading = lookup_preintegrated_transfer(optical_depth, transfer_func)
            
        # Optional stochastic single scattering [S1]
        if enable_single_scattering:
            shading += compute_single_scattering(x, ray_dir, subset)
            
        # 6. Accumulation [G]
        color += shading * alpha * (1.0 - alpha)
        alpha += (1.0 - alpha) * shading.alpha
        t += step_length
        
        if alpha >= 1.0:
            break
            
    return color
```

## Operating characteristics

*   **Time Complexity**: Depends on the number of particles sampled per step. By sampling a subset, it reduces the complexity from $O(N)$ per step to $O(k)$ where $k \ll N$. The overall complexity is $O(R \cdot S \cdot k)$ where $R$ is the number of rays, $S$ is the number of steps per ray, and $k$ is the number of sampled particles per step [S1].
*   **Memory Complexity**: $O(N)$ for storing particles and the grid. The grid adds minimal overhead as it only stores references [S1].
*   **Performance**: Scales continuously from fast, interactive previews to high-quality rendering by adjusting the number of sampled particles [S1].
*   **Strengths**: Minimal preprocessing, no intermediate grid representation, handles large datasets, supports multi-phase flows, efficient GPU implementation with coalesced memory access [S1].
*   **Limitations**: Stochastic sampling introduces noise; requires careful tuning of the importance measure and number of samples for quality [S1]. Pre-integration is limited to homogeneous segments [S1].
*   **When to Use**: Large SPH datasets where preprocessing time and memory are constraints, and interactive or semi-interactive visualization is desired. Particularly suitable for multi-phase flows [S1].

## Background

*   **Origin**: The method was introduced in the paper "Stochastic Volume Rendering of Multi-Phase SPH Data" [S1].
*   **History**: It builds on previous work in SPH visualization and stochastic rendering. It addresses the computational complexity of rendering large, unstructured particle data by avoiding intermediate representations and using importance sampling [S1].
*   **Usage**: Used in scientific visualization for fluid dynamics, particularly for multi-phase flows like gas-liquid interactions [S1].

## Sources

[S1] Stochastic Volume Rendering of Multi‐Phase SPH Data, https://doi.org/10.1111/cgf.14121, fetched 2026-10-27
