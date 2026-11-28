# RAYTHEIA

Also called: raytheia

## Overview
RAYTHEIA is a high-performance reverse ray-tracing algorithm designed to efficiently solve three-dimensional direction-dependent radiative transfer equations in astronomical simulations, specifically for photodissociation regions (PDRs) [S1, unofficial]. It solves the problem of computing direction-dependent column densities and optical depths for large numbers of receiving grid cells by emitting rays from targets towards sources, rather than from sources to receivers [S1, unofficial]. Unlike forward ray-tracing or full-scattering Monte Carlo codes, RAYTHEIA is optimized for absorption-dominated regimes where scattering is negligible, leveraging a dual-grid framework with a leaf-only linear-octree and DDA traversal to achieve near-ideal linear speed-up [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A native simulation mesh (source grid) containing physical properties (density $\rho$, emission coefficient $j_\nu$, absorption opacity $\kappa_{\nu,a}$); an Adaptive Mesh Refinement (AMR) Cartesian contribution grid constructed for ray-walking; HEALPix ray directions defined by refinement level $\ell$.
*   **Output:** Integrated intensity $I_\nu$ or optical depth $\tau_\nu$ along each ray direction for every cell in the source grid.

**Data Structures:**
1.  **Native Simulation Mesh:** The base grid where physical data resides.
2.  **AMR Contribution Grid:** A hierarchical structure built from the native mesh.
    *   **Structure:** Leaf-only linear-octree. This structure stores only the leaf nodes of the octree, reducing memory overhead compared to storing internal nodes [S1, unofficial].
    *   **Indexing:** Morton Code (Z-order curve) indexing is used to map 3D spatial coordinates to 1D linear indices for fast leaf cell lookup during traversal [S1, unofficial].
    *   **Layout:** The linear array contains leaf node data. Each leaf represents a voxel in the AMR grid. The Morton code allows determining parent-child relationships and spatial locality without explicit pointers [S1, unofficial].

**Governing Equation:**
The algorithm solves the simplified radiative transfer equation neglecting scattering ($\kappa_{\nu,s} \approx 0$) [S1, unofficial]:
$$ \mathbf{n} \cdot \nabla I_\nu = -\kappa_{\nu,a} \rho I_\nu + j_\nu \rho $$
where:
*   $\nu$: specific frequency.
*   $I_\nu$: specific intensity.
*   $\kappa_{\nu,a}$: absorption opacity.
*   $\rho$: mass density.
*   $j_\nu$: emission coefficient.
*   $\mathbf{n}$: unit vector in the direction of propagation.

The solution along a ray yields the attenuation law $I_\nu(\tau_\nu) = I_{\nu,0} e^{-\tau_\nu}$, where $\tau_\nu$ is the accumulated optical depth [S1, unofficial].

**Parameters:**
*   **Ray Count:** $N_{rays} = 12 \times 4^\ell$, where $\ell$ is the HEALPix level of refinement [S1, unofficial].
*   **Ray Direction:** Rays are emanated from every grid cell in the source grid towards the entire domain using HEALPix directions [S1, unofficial]. HEALPix ensures every ray traces an identical solid angle [S1, unofficial].

**Algorithm Steps:**
1.  **Domain Decomposition:** The simulation domain is decomposed into chunks for distributed parallel processing using MPI [S1, unofficial].
2.  **Ray Emission:** For each cell in the source grid, rays are generated in all $N_{rays}$ HEALPix directions [S1, unofficial].
3.  **Path Determination (Traversal):** For each ray, a parametric octree traversal method (Digital Differential Analyzer - DDA) is employed to determine the path through the AMR contribution grid [S1, unofficial].
    *   The DDA algorithm steps through the octree, identifying the sequence of leaf cells intersected by the ray [S1, unofficial].
    *   Morton Code indexing is used to quickly locate the leaf node corresponding to the current voxel in the traversal path [S1, unofficial].
4.  **Contribution Accumulation (Ray-walk):** For each leaf cell intersected by the ray, the integral along the ray is computed [S1, unofficial].
    *   The **Slab Method** is used to analytically compute the theoretical path length $L$ of the ray through the current grid cell [S1, unofficial].
    *   The optical depth contribution $\Delta \tau_\nu$ for the cell is calculated as $\Delta \tau_\nu = \kappa_{\nu,a} \rho L$ [S1, unofficial].
    *   The intensity is updated using the attenuation law: $I_{\nu, new} = I_{\nu, old} e^{-\Delta \tau_\nu} + \frac{j_\nu \rho}{\kappa_{\nu,a} \rho} (1 - e^{-\Delta \tau_\nu})$ [S1, unofficial]. Note: If emission is neglected or treated via escape probability, the second term may be simplified or omitted depending on the specific implementation within 3d-pdr [S1, unofficial].
5.  **Parallel Communication:** A chunk-to-chunk communication strategy is used to handle data dependencies between MPI ranks during the accumulation phase [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def raytheia_traverse(source_grid, amr_octree, healpix_level):
    """
    Main loop for RAYTHEIA reverse ray-tracing.
    source_grid: 3D array of physical properties (rho, kappa, j)
    amr_octree: Leaf-only linear-octree with Morton indexing
    healpix_level: Integer l defining ray directions
    """
    # 1. Define Ray Directions
    # N_rays = 12 * 4^l
    # Directions are generated using HEALPix scheme
    directions = generate_healpix_directions(healpix_level)
    n_rays = len(directions)
    
    # 2. Initialize Output Storage
    # For each cell in source_grid, store accumulated intensity/optical depth
    # for each direction
    results = zeros_like(source_grid, shape=(..., n_rays))
    
    # 3. Parallel Loop over Source Grid Cells
    # Domain decomposition handled by MPI chunks
    for cell in source_grid.cells:
        # Get physical properties at this cell
        rho = cell.density
        kappa = cell.absorption_opacity
        j = cell.emission_coefficient
        
        # Initial intensity at cell boundary (could be 0 or background)
        I_nu = 0.0 
        
        # 4. Traverse each Ray Direction
        for dir_idx, n_vec in enumerate(directions):
            # Initialize DDA state for ray starting at cell center
            # with direction n_vec
            dda_state = init_dda(cell.center, n_vec, amr_octree.bounds)
            
            # 5. DDA Traversal
            # Step through the octree to find intersected leaf cells
            while dda_state.has_next_cell():
                # Get next leaf cell index using Morton Code lookup
                leaf_idx = dda_state.get_next_leaf_index()
                
                # Retrieve cell properties from AMR grid
                # Note: AMR grid may have different resolution than source grid
                # Interpolation or averaging may be needed if resolutions differ
                # [G]: Assuming AMR grid values are directly accessible or interpolated
                cell_rho = amr_octree.get_density(leaf_idx)
                cell_kappa = amr_octree.get_opacity(leaf_idx)
                cell_j = amr_octree.get_emission(leaf_idx)
                
                # 6. Slab Method: Calculate Path Length
                # Compute intersection interval [t_start, t_end] of ray with current voxel
                t_start, t_end = dda_state.get_intersection_interval()
                path_length = t_end - t_start
                
                # 7. Accumulate Contribution
                # Optical depth increment
                delta_tau = cell_kappa * cell_rho * path_length
                
                # Update intensity using attenuation law
                # I_new = I_old * exp(-delta_tau) + (j/kappa) * (1 - exp(-delta_tau))
                # [G]: If j is negligible or handled separately, simplify
                if cell_kappa > 0:
                    source_term = cell_j / cell_kappa
                    I_nu = I_nu * exp(-delta_tau) + source_term * (1 - exp(-delta_tau))
                else:
                    I_nu = I_nu * exp(-delta_tau)
                
                # Advance DDA
                dda_state.advance()
            
            # Store result for this direction
            results[cell.index, dir_idx] = I_nu
            
    return results
```

**Edge Cases and Pitfalls:**
*   **Scattering:** RAYTHEIA neglects diffuse radiation (scattering). This is a valid approximation for many PDR calculations but may alter molecular abundances and line emission if scattering is significant [S1, unofficial].
*   **Resolution:** The algorithm demonstrates accuracy and convergence even at low angular resolutions, but higher HEALPix levels ($\ell$) increase $N_{rays}$ quadratically ($4^\ell$), impacting performance [S1, unofficial].
*   **Dual-Grid Consistency:** The native simulation mesh and AMR contribution grid must be consistent. The AMR grid is constructed for efficient ray-walking, implying potential interpolation or data transfer between the two grids [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of rays and the number of leaf cells intersected by rays. The DDA traversal is efficient, and Morton Code lookup is $O(1)$ or $O(\log N)$ depending on implementation [S1, unofficial].
*   **Memory Complexity:** Reduced by using a leaf-only linear-octree structure, storing only leaf nodes [S1, unofficial].
*   **Performance:** Achieves exceptional, near-ideal linear speed-up ratio with hybrid MPI/OpenMP parallelization [S1, unofficial].
*   **Strengths:** High performance, low memory overhead, suitable for large-scale astronomical simulations (e.g., 512^3 grid cells) [S1, unofficial]. Optimized for direction-dependent integrations in PDR chemistry [S1, unofficial].
*   **Limitations:** Neglects scattering, making it unsuitable for dust continuum or scattered-light problems where full-scattering Monte Carlo codes (like SKIRT) are required [S1, unofficial].
*   **When to Use:** When modeling PDR chemistry and thermal balance in 3D turbulent clouds with high spatial resolution, requiring direction-dependent column densities and optical depths [S1, unofficial].
*   **When Not to Use:** When scattering is significant or when full radiative transfer with diffuse radiation is required [S1, unofficial].

## Background
RAYTHEIA was developed to address the computational demands of 3D photodissociation region (PDR) chemistry in turbulent, star-forming clouds [S1, unofficial]. It integrates with the 3d-pdr code to solve complex chemistry and radiation transfer [S1, unofficial]. The algorithm builds on reverse ray-tracing approaches used in previous works (e.g., Bisbas et al. 2012, Clark et al. 2012) and incorporates techniques from computer graphics such as DDA traversal (Amanatides et al. 1987) and the slab method (Kay & Kajiya 1986) [S1, unofficial]. It was presented in a paper by researchers from Zhejiang Laboratory and the University of Duisburg-Essen [S1, unofficial].

## Sources
[S1] RAYTHEIA: A high-performance ray-tracing algorithm for three-dimensional direction-dependent equations in astronomical simulations, https://arxiv.org/pdf/2605.09882, fetched 2026-10-24 (unofficial)
