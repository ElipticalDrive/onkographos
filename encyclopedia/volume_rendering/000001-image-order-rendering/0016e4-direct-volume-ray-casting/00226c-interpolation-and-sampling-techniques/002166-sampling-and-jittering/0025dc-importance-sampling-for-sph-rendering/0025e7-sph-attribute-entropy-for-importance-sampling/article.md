# SPH Attribute Entropy for Importance Sampling

Also called: Data importance precomputation, Entropy-based importance

## Overview
SPH Attribute Entropy for Importance Sampling is a precomputation technique that assigns sampling probabilities to SPH particles based on the local information complexity of their attributes and the presence of phase interfaces [S1]. It solves the problem of high-frequency shading errors and subsampling artifacts at phase boundaries by ensuring that regions with high attribute variation or discontinuous phase transitions are sampled more densely [S1]. It differs from naive random discarding by using a structured, data-driven importance grid that accounts for the spatial support of the SPH kernel, and it differs from its sibling [Stochastic Volume Rendering of Multi-Phase SPH Data](3638-stochastic-volume-rendering-of-multi-phase-sph-data.md) by explicitly computing entropy to quantify attribute variation rather than relying solely on stochastic selection heuristics.

## How it works
The method operates in two phases: a precomputation phase that builds an importance grid from the particle data, and a runtime phase that queries this grid during ray marching to determine particle sampling probabilities.

### Inputs and Outputs
*   **Inputs:**
    *   Particle set $\mathcal{P}$, where each particle $i$ has position $\mathbf{x}_i$, phase $\pi_i$, and attribute values for the visualized SPH attribute $A$.
    *   Rendering SPH kernel function with finite support radius $h$.
    *   Importance grid resolution and bounds covering the spatial extent of the SPH data.
    *   Number of histogram bins $N_{\text{bins}}$ for entropy calculation.
    *   Dilation kernel width (in grid cells) and number of dilation iterations.
*   **Outputs:**
    *   Importance grid storing scalar importance values $p_i(\mathbf{x})$ for each cell.
    *   Runtime sampling probability $p(\mathbf{x})$ for any position $\mathbf{x}$ during rendering.

### Data Structures
1.  **Importance Grid:** A 3D regular grid where each cell stores a single floating-point value representing the data importance $p_i(\mathbf{x}_c)$ at the cell center $\mathbf{x}_c$. The grid resolution is chosen to cover the bounding box of the particle set.
2.  **Histogram:** A temporary array of size $N_{\text{bins}}$ used to compute the entropy of attribute values for particles within a local neighborhood.

### Precomputation Steps
The precomputation determines the initial importance for each grid cell and then filters these values to account for the SPH kernel's spatial support and interface detection.

1.  **Initialize Grid:** Create an importance grid of sufficient resolution to cover the particle domain. Initialize all cells to 0.
2.  **Compute Initial Importance:** For each grid cell $c$ with center $\mathbf{x}_c$:
    a. Identify the set of particles $\mathcal{P}(\mathbf{x}_c)$ lying within this cell.
    b. If $\mathcal{P}(\mathbf{x}_c)$ is empty, set $p_i(\mathbf{x}_c) = 0$.
    c. If $\mathcal{P}(\mathbf{x}_c)$ contains particles with different phases (i.e., $\exists i, j \in \mathcal{P}(\mathbf{x}_c)$ such that $\pi_i \neq \pi_j$), set $p_i(\mathbf{x}_c) = 1$.
    d. Otherwise, if all particles in $\mathcal{P}(\mathbf{x}_c)$ have the same phase, compute the entropy of attribute $A$ for these particles:
        i. Bin the attribute values of particles in $\mathcal{P}(\mathbf{x}_c)$ into a normalized histogram $h(b)$ with $N_{\text{bins}}$ bins.
        ii. Compute the entropy:
        $$ \mathbb{H}_{\mathcal{P}(\mathbf{x}_c)}(A) = - \sum_{b=0}^{N_{\text{bins}}-1} h(b) \log_2 h(b) $$
        iii. Normalize the entropy to be independent of histogram size:
        $$ \overline{\mathbb{H}}_{\mathcal{P}(\mathbf{x}_c)}(A) = \frac{2^{\mathbb{H}_{\mathcal{P}(\mathbf{x}_c)}(A)}}{N_{\text{bins}}} $$
        iv. Set $p_i(\mathbf{x}_c) = \overline{\mathbb{H}}_{\mathcal{P}(\mathbf{x}_c)}(A)$.
3.  **Filter with Maximum Dilation:** Apply a 3D maximum dilation filter to the importance grid to propagate importance values to neighboring cells, compensating for the fact that SPH approximation at a sampling point depends on particles in neighboring cells.
    a. Define a dilation kernel with width equal to the number of grid cells covered by the finite support of the SPH kernel function used in rendering.
    b. For each cell in the grid, update its importance value to the maximum importance value found in any cell within the dilation kernel.
    c. **Interface Detection:** During dilation, if more than half of the cells in the dilation kernel do not contain any particles, the cell is expected to contain a free-surface interface. In this case, set the cell's importance $p_i(\mathbf{x}_c)$ to 1.
4.  **Iterate Dilation:** Repeat the dilation step for a user-specified number of iterations. In practice, 1 or 2 iterations are sufficient [S1].

### Runtime Sampling
During ray marching, for a sampling position $\mathbf{x}$:
1.  Query the precomputed importance value $p_i(\mathbf{x})$ from the importance grid cell containing $\mathbf{x}$.
2.  Combine $p_i(\mathbf{x})$ with the view-dependent transmittance factor $p_t(\mathbf{x})$ to determine the particle sampling probability $p(\mathbf{x})$ according to Equation 9 of the parent article [S1]. Note: Equation 9 is not provided in the sources; the combination is typically a weighted product or sum normalized to [0,1]. [G] A common implementation is $p(\mathbf{x}) = \frac{p_i(\mathbf{x}) \cdot p_t(\mathbf{x})}{\max(p_i(\mathbf{x}) \cdot p_t(\mathbf{x}))}$ or similar normalization, but the exact formula depends on the parent article's definition of $p_t(\mathbf{x})$.
3.  Use $p(\mathbf{x})$ to stochastically select particles for the SPH approximation at $\mathbf{x}$.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def compute_entropy_histogram(attribute_values, n_bins):
    """
    Computes normalized histogram and entropy for a set of attribute values.
    Returns normalized entropy.
    """
    # Create histogram
    hist, _ = np.histogram(attribute_values, bins=n_bins, range=(min(attribute_values), max(attribute_values)))
    # Normalize histogram
    hist = hist / np.sum(hist)
    # Compute entropy
    # Handle log(0) by ignoring bins with 0 probability
    entropy = -np.sum(hist[hist > 0] * np.log2(hist[hist > 0]))
    # Normalize entropy
    normalized_entropy = (2 ** entropy) / n_bins
    return normalized_entropy

def precompute_importance_grid(particles, grid_resolution, grid_bounds, n_bins, kernel_support_cells, dilation_iterations):
    """
    Precomputes the importance grid for SPH attribute entropy sampling.
    
    Args:
        particles: List of particle objects with attributes: pos (x,y,z), phase, attr_value
        grid_resolution: Tuple (nx, ny, nz) for grid dimensions
        grid_bounds: Tuple ((min_x, max_x), (min_y, max_y), (min_z, max_z))
        n_bins: Number of histogram bins for entropy calculation
        kernel_support_cells: Number of grid cells covered by SPH kernel support
        dilation_iterations: Number of dilation passes
        
    Returns:
        importance_grid: 3D numpy array of importance values
    """
    nx, ny, nz = grid_resolution
    importance_grid = np.zeros((nx, ny, nz))
    
    # Map particles to grid cells
    # Assume particles have a method or attribute to determine their grid cell index
    # For simplicity, we assume a helper function get_cell_index exists
    cell_particles = {} # Key: (i,j,k), Value: list of particles
    
    for p in particles:
        i, j, k = get_cell_index(p.pos, grid_bounds, grid_resolution)
        if (i, j, k) not in cell_particles:
            cell_particles[(i, j, k)] = []
        cell_particles[(i, j, k)].append(p)
        
    # Step 2: Compute initial importance
    for i in range(nx):
        for j in range(ny):
            for k in range(nz):
                if (i, j, k) in cell_particles:
                    cell_p_list = cell_particles[(i, j, k)]
                    phases = [p.phase for p in cell_p_list]
                    
                    if len(cell_p_list) == 0:
                        importance_grid[i, j, k] = 0
                    elif len(set(phases)) > 1:
                        importance_grid[i, j, k] = 1.0
                    else:
                        attr_values = [p.attr_value for p in cell_p_list]
                        importance_grid[i, j, k] = compute_entropy_histogram(attr_values, n_bins)
                else:
                    importance_grid[i, j, k] = 0
                    
    # Step 3 & 4: Filter with Maximum Dilation
    # Define kernel size
    kernel_size = kernel_support_cells
    half_kernel = kernel_size // 2
    
    for iteration in range(dilation_iterations):
        new_grid = np.copy(importance_grid)
        for i in range(nx):
            for j in range(ny):
                for k in range(nz):
                    # Define kernel bounds
                    i_min = max(0, i - half_kernel)
                    i_max = min(nx - 1, i + half_kernel)
                    j_min = max(0, j - half_kernel)
                    j_max = min(ny - 1, j + half_kernel)
                    k_min = max(0, k - half_kernel)
                    k_max = min(nz - 1, k + half_kernel)
                    
                    # Check for interface: count empty cells in kernel
                    empty_count = 0
                    max_importance = 0.0
                    for ii in range(i_min, i_max + 1):
                        for jj in range(j_min, j_max + 1):
                            for kk in range(k_min, k_max + 1):
                                if importance_grid[ii, jj, kk] == 0:
                                    empty_count += 1
                                else:
                                    max_importance = max(max_importance, importance_grid[ii, jj, kk])
                                    
                    total_cells = (i_max - i_min + 1) * (j_max - j_min + 1) * (k_max - k_min + 1)
                    
                    # Interface detection
                    if empty_count > total_cells / 2:
                        new_grid[i, j, k] = 1.0
                    else:
                        new_grid[i, j, k] = max_importance
                        
        importance_grid = new_grid
        
    return importance_grid

def get_cell_index(pos, grid_bounds, grid_resolution):
    """
    Helper function to map a 3D position to a grid cell index.
    """
    x, y, z = pos
    (min_x, max_x), (min_y, max_y), (min_z, max_z) = grid_bounds
    nx, ny, nz = grid_resolution
    
    dx = (max_x - min_x) / nx
    dy = (max_y - min_y) / ny
    dz = (max_z - min_z) / nz
    
    i = int((x - min_x) / dx)
    j = int((y - min_y) / dy)
    k = int((z - min_z) / dz)
    
    # Clamp to grid bounds
    i = max(0, min(nx - 1, i))
    j = max(0, min(ny - 1, j))
    k = max(0, min(nz - 1, k))
    
    return (i, j, k)
```

### Parameters
*   **$N_{\text{bins}}$:** Number of histogram bins for entropy calculation. Not given in the sources. [G] A typical value might be 256 or 512, depending on the attribute range and desired precision.
*   **Dilation kernel width:** Equal to the number of grid cells covered by the finite support of the SPH kernel function used in rendering.
*   **Dilation iterations:** User-specified number of iterations for the filtering process; 1 or 2 iterations are sufficient in practice [S1].

### Edge Cases and Pitfalls
*   **Phase Interface Discontinuity:** The approximation of phases is discontinuous, leading to edge cases where false distinct phases may be assigned to a sampling point, causing poor surface shading due to high-frequency specular highlights and high opacities [S1].
*   **Subsampling Errors at Interfaces:** Regions with different phases contain phase interfaces provoking surface shading; small SPH subsampling errors lead to poor surface shading due to dependency on normals and high opacities [S1].
*   **Empty Cells:** Cells with no particles are assigned importance 0, which is handled correctly by the interface detection logic during dilation.

## Subtopics

### [Stratified Stochastic Particle Sampling](4711-stratified-stochastic-particle-sampling.md)
Stratified Stochastic Particle Sampling reduces overall sampling error in SPH rendering by selecting well-distributed particle subsets within an acceleration structure's cells. It differs from naive random discarding by enforcing a structured selection process based on cell and position importance, ensuring that the statistical properties of the particle neighborhood are preserved more accurately during ray marching.

## Operating characteristics
*   **Time Complexity:** Precomputation is $O(N \cdot K)$, where $N$ is the number of particles and $K$ is the number of grid cells, plus $O(K \cdot W^3 \cdot I)$ for the dilation, where $W$ is the kernel width in cells and $I$ is the number of iterations. Runtime sampling is $O(1)$ per sampling point for grid query, plus the cost of stochastic particle selection.
*   **Memory Complexity:** $O(K)$ for the importance grid, where $K$ is the number of grid cells.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales with the number of particles and grid resolution. Higher grid resolution increases memory and precomputation time but may improve accuracy.
*   **Strengths:** Reduces sampling errors at phase interfaces and regions with high attribute variation. Precomputation allows for efficient runtime sampling.
*   **Limitations:** Requires precomputation step. Accuracy depends on grid resolution and kernel support estimation.
*   **When to Use:** When rendering SPH data with multiple phases or significant attribute variation, and when surface shading quality is critical.
*   **When Not to Use:** When rendering simple, single-phase SPH data with uniform attributes, or when precomputation time is prohibitive.

## Background
The method was introduced in the paper "Stochastic Volume Rendering of Multi-Phase SPH Data" [S1]. It addresses the challenges of rendering SPH data with multiple phases, where traditional uniform sampling can lead to artifacts at phase boundaries. The use of entropy to measure attribute variation is based on information theory [Sha48], and the normalization of entropy follows Wei et al. [WDS18].

## Sources
[S1] Stochastic Volume Rendering of Multi‐Phase SPH Data, https://doi.org/10.1111/cgf.14121, fetched 2026-10-27
