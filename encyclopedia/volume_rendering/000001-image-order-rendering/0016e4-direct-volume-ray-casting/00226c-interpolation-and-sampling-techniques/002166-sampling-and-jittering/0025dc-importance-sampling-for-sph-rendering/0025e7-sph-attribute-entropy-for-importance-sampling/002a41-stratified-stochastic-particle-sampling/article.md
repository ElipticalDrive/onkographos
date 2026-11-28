# Stratified Stochastic Particle Sampling

Also called: Stratification of particle samples

## Overview
Stratified Stochastic Particle Sampling reduces overall sampling error in SPH rendering by selecting well-distributed particle subsets within an acceleration structure's cells [S1]. It differs from naive random discarding by enforcing a structured selection process based on cell and position importance, ensuring that the statistical properties of the particle neighborhood are preserved more accurately during ray marching [S1].

## How it works
**Inputs:**
* Particle set $\mathcal{P}$ organized in a uniform grid acceleration structure [S1].
* Data importance grid $p_i(\mathbf{x}_c)$, precomputed for each cell center $\mathbf{x}_c$ [S1].
* Sampling position $\mathbf{x}$ within the volume [G].
* Probability bounds $p_{\min}$ and $p_{\max}$ [S1].

**Outputs:**
* A subset of particles chosen from the relevant acceleration structure cells to evaluate the SPH approximation at $\mathbf{x}$ [S1].

**Data Structures:**
* The acceleration structure is a uniform grid storing particles, aligned with the data importance grid [S1].
* Both grids share equal cell sizes and perfect overlap, ensuring overlapping cells share the same center point $\mathbf{x}_c$ [S1].

**Parameters:**
* $p_{\min}$ and $p_{\max}$ define the sampling probability bounds [S1].
* Grid resolution determines cell size; optimal cell width equals the kernel function support for efficient particle access [S1].
* Finer grids increase the stratification factor but may decrease performance [S1].

**Algorithm:**
1. Identify all cells in the acceleration structure that contribute to the SPH approximation at $\mathbf{x}$ [G]. Let $\mathcal{C}(\mathbf{x})$ be this set [G].
2. For each cell center $\mathbf{x}_c \in \mathcal{C}(\mathbf{x})$:
   a. Retrieve the particle count $|\mathcal{P}(\mathbf{x}_c)|$ in the cell [S1].
   b. Retrieve the cell importance $p_i(\mathbf{x}_c)$ [S1].
   c. Compute the position importance $p_i(\mathbf{x})$ for the sampling location [G].
   d. Calculate the target number of particles $n$ using $n = p_{\min} + (p_{\max} - p_{\min}) p_i(\mathbf{x}_c) p_i(\mathbf{x}) |\mathcal{P}(\mathbf{x}_c)|$ [S1].
   e. Select $\lfloor n \rfloor$ particles from $\mathcal{P}(\mathbf{x}_c)$ [S1].
   f. Select one additional particle with probability $n - \lfloor n \rfloor$ [S1]. (The source's "$(n - n)$" is a typographical error for the fractional part.)
3. Aggregate the selected particles from all cells in $\mathcal{C}(\mathbf{x})$ to perform the SPH approximation [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def stratified_stochastic_sampling(particles_grid, importance_grid, x, p_min, p_max, kernel_radius):
    """
    Selects particles for SPH approximation at position x using stratification.
    """
    selected_particles = []
    cell_size = particles_grid.cell_size
    
    # 1. Identify contributing cells for SPH approximation at x
    # [G] Standard SPH kernel support determines which grid cells overlap the kernel sphere.
    start_idx = (math.floor((x[0] - kernel_radius) / cell_size), math.floor((x[1] - kernel_radius) / cell_size), math.floor((x[2] - kernel_radius) / cell_size))
    end_idx = (math.floor((x[0] + kernel_radius) / cell_size), math.floor((x[1] + kernel_radius) / cell_size), math.floor((x[2] + kernel_radius) / cell_size))
    
    # Precompute position importance p_i(x)
    # [G] Exact formula for position importance is not provided in the sources.
    position_importance = 1.0 
    
    for i in range(start_idx[0], end_idx[0] + 1):
        for j in range(start_idx[1], end_idx[1] + 1):
            for k in range(start_idx[2], end_idx[2] + 1):
                xc = (i + 0.5) * cell_size
                cell_particles = particles_grid.get_particles_at_center(xc)
                N_c = len(cell_particles)
                
                if N_c == 0:
                    continue
                    
                cell_importance = importance_grid.get_value_at_center(xc)
                
                # 2d. Calculate target number n
                n = p_min + (p_max - p_min) * cell_importance * position_importance * N_c
                
                # 2e & 2f. Select integer part
                n_int = math.floor(n)
                if n_int > 0:
                    indices = random.sample(range(N_c), min(n_int, N_c))
                    selected_particles.extend([cell_particles[idx] for idx in indices])
                    
                # 2g. Probabilistic additional particle
                remainder = n - n_int
                if remainder > 0 and n_int < N_c:
                    if random.random() < remainder:
                        remaining_indices = [idx for idx in range(N_c) if idx not in indices]
                        if remaining_indices:
                            selected_particles.append(cell_particles[random.choice(remaining_indices)])
                            
    return selected_particles
```

**Edge Cases and Pitfalls:**
* Multiple Cell Access: At each sampling position, multiple cells are accessed. The position importance $p_i(\mathbf{x})$ remains constant across all accessed cells, but the cell importance $p_i(\mathbf{x}_c)$ can differ between these cells during a single SPH approximation [S1].
* Grid Alignment: The method assumes the acceleration structure and the data importance grid have equal cell sizes and perfect overlap, sharing the same center point $\mathbf{x}_c$ [S1].
* Performance vs. Stratification: Using a finer grid resolution increases the stratification factor but can decrease performance. The optimal cell width is equal to the kernel function support [S1].

## Operating characteristics
* Time Complexity: Depends on the number of contributing cells and particles per cell [S1].
* Memory Complexity: Requires storage for the uniform grid acceleration structure and the data importance grid [S1].
* Strengths: Reduces overall sampling error by ensuring well-distributed samples in the neighborhood [S1]. It avoids the variance associated with purely random discarding of particles [S1].
* Limitations: Performance can degrade if the grid resolution is too fine [S1]. It requires precomputation of importance values [S1].
* When to use: Used in SPH rendering to reduce sampling error in neighborhoods [S1].

## Background
The method is described in the context of stochastic volume rendering of multi-phase SPH data [S1]. It builds upon the concept of importance sampling using SPH attribute entropy [3626-sph-attribute-entropy-for-importance-sampling.md]. The technique was introduced to handle the high-frequency nature of surface shading and phase interfaces in SPH rendering [S1].

## Sources
[S1] Stochastic Volume Rendering of Multi‐Phase SPH Data, https://doi.org/10.1111/cgf.14121, fetched 2026-10-27
