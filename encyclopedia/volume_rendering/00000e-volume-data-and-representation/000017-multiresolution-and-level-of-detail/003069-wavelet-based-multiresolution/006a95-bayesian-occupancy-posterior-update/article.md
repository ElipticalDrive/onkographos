# Bayesian occupancy posterior update
Also called: Log-odds occupancy integration

## Overview
This method performs the incremental update of a map's Bayesian occupancy posterior when a new beam endpoint measurement arrives [S1]. It replaces the per-voxel update rule of the TSDF integrator while sharing the grow-then-inject scaffold [S2, unofficial]. It is applied in the wavemap framework for robotics volumetric mapping to update occupancy information [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `Grid`: The current sparse grid structure.
    *   `ℓ`: Log-odds sidecar array of length $|Grid|$.
    *   `Π`, `ρ`, `κ`: Parameters from the parent algorithm (grow-then-inject scaffold) [S2, unofficial].
    *   `ℓ_hit`: Bayesian increment for hit voxels.
    *   `ℓ_miss`: Bayesian increment for miss voxels.
    *   `[ℓ_min, ℓ_max]`: Clamp band for log-odds values [S2, unofficial].
    *   `R`: Set of rays derived from LiDAR endpoints.
*   **Outputs:**
    *   `Grid'`: Updated grid structure.
    *   `ℓ'`: Updated log-odds sidecar [S2, unofficial].

**Data Structures:**
*   **Log-odds sidecar (`ℓ`):** An array storing the log-odds ratio for each voxel in the grid. The log-odds $L_p$ is defined as the logarithm of the odds of occupancy [S1].

**Formulas:**
The Bayesian occupancy posterior update is performed in the log-odds domain to convert the product of probabilities into a sum [S1]:
$$L_p(m_x | z_{1:t}) = L_p(m_x | z_{1:t-1}) + L_s(m_x | z_t)$$
Where:
*   $m_x$ is the occupancy state at point $x$ in the map $m$ [S1].
*   $z_{1:t}$ is the sequence of beam endpoint measurements up to time $t$ [S1].
*   $L_p(m_x | z_{1:t-1})$ is the prior log-odds (previous state) [S1].
*   $L_s(m_x | z_t)$ is the sensor model log-odds increment for the new measurement $z_t$ [S1].

**Steps:**
1.  **Build Topology Shell:** Create a shell of voxels from LiDAR endpoints expanded by a ball of radius $\rho$:
    $$\text{shell} \leftarrow \text{voxelsToGrid}(\text{LiDAREndpoints}(P) \oplus \text{ball}(\rho))$$ [S2, unofficial].
2.  **Merge Grids:** Merge the current grid with the shell to ensure all relevant voxels exist:
    $$\text{Grid}' \leftarrow \text{mergeGrids}(\text{Grid}, \text{shell})$$ [S2, unofficial].
3.  **Inject Sidecar:** Inject the previous log-odds sidecar into the new grid structure, initializing new voxels to 0:
    $$\ell' \leftarrow \text{inject}(\text{Grid} \rightarrow \text{Grid}', \ell, \text{init}=0)$$ [S2, unofficial].
4.  **Ray Walk and Atomic Update:** For each ray $r \in R$ in parallel via HDDA ray-walk:
    *   For each voxel $v$ traversed by $r$:
        *   Determine the log-odds increment:
            $$\text{inc} \leftarrow v.\text{isEndpoint}(v) ? \ell_{\text{hit}} : \ell_{\text{miss}}$$ [S2, unofficial].
        *   Atomically add to the sidecar:
            $$\text{atomicAdd}(\ell'[v], \text{inc})$$ [S2, unofficial].
5.  **Clamp:** Clamp the log-odds values to the band $[\ell_{\min}, \ell_{\max}]$:
    $$\ell' \leftarrow \text{clamp}(\ell', \ell_{\min}, \ell_{\max})$$ [S2, unofficial].
6.  **Return:** Return $\text{Grid}'$ and $\ell'$ [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def log_odds_occupancy_integration(Grid, l, Pi, rho, kappa, l_hit, l_miss, l_min, l_max, R):
    # 1. Build topology shell from LiDAR endpoints expanded by ball of radius rho
    # Note: LiDAREndpoints(P) assumes P is the point cloud associated with rays R
    endpoints = get_lidar_endpoints(R)
    shell = voxels_to_grid(dilate_with_ball(endpoints, rho))
    
    # 2. Merge grids to ensure topology coverage
    Grid_prime = merge_grids(Grid, shell)
    
    # 3. Inject previous log-odds sidecar, initializing new voxels to 0
    l_prime = inject_sidecar(Grid, Grid_prime, l, init=0)
    
    # 4. Parallel ray walk
    # Each thread handles one ray r from R
    for r in parallel(R):
        # Walk through voxels traversed by ray r using HDDA
        voxels_traversed = hdda_ray_walk(r)
        
        for v in voxels_traversed:
            # Determine if this voxel is the endpoint of the ray
            is_endpoint = v.is_endpoint_of(r)
            
            # Select increment based on hit or miss
            if is_endpoint:
                inc = l_hit
            else:
                inc = l_miss
            
            # Atomic addition to the sidecar
            atomic_add(l_prime[v], inc)
            
    # 5. Clamp log-odds values
    l_prime = clamp(l_prime, l_min, l_max)
    
    # 6. Return updated structures
    return Grid_prime, l_prime
```

**Parameters:**
*   $\ell_{\text{hit}}$: Bayesian increment for hit voxels. Value not given in the sources.
*   $\ell_{\text{miss}}$: Bayesian increment for miss voxels. Value not given in the sources.
*   $[\ell_{\min}, \ell_{\max}]$: Clamp band. Value not given in the sources.
*   $\rho$: Radius of the ball used for expanding LiDAR endpoints. Value not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** Log-odds arithmetic avoids underflow/overflow issues associated with direct probability multiplication, but clamping is required to prevent extreme values [S2, unofficial].
*   **Boundaries:** The `clamp` operation ensures values stay within $[\ell_{\min}, \ell_{\max}]$ [S2, unofficial].
*   **Degenerate Input:** If a ray has no voxels or endpoints, the loop simply does not execute for that ray [S2, unofficial].

## Subtopics

### Projecting Detail Coefficients
Projects detail coefficients to update the wavelet representation. Choose when detail updates are needed. [Link](1802-projecting-detail-coefficients.md)

### Wavelet Tree Overview Map
Provides a low-resolution overview of the wavelet tree. Choose for visualization or coarse culling. [Link](5354-wavelet-tree-overview-map.md)

### Wavelet Tree
The hierarchical data structure for wavelet coefficients. Choose as the base representation. [Link](5390-wavelet-tree.md)

### Cardinal B-spline wavelets
A specific type of wavelet basis. Choose for smoothness properties. [Link](11756-cardinal-b-spline-wavelets.md)

### Multi-Resolution Analysis (MRA) conditions
Conditions for valid multi-resolution representations. Choose to ensure theoretical consistency. [Link](23140-multi-resolution-analysis-mra-conditions.md)

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels traversed by rays and the number of rays [S2, unofficial].
*   **Memory Complexity:** Linear with respect to the number of voxels in the grid [S2, unofficial].
*   **Measured Performance:** Occupancy is the smallest integrator (~430 lines in one custom kernel) [S2, unofficial].
*   **Strengths:** Efficient incremental update; shares scaffold with other integrators; suitable for robotics mapping [S1][S2, unofficial].
*   **Limitations:** Requires atomic operations for parallel updates; sensitive to parameter choices ($\ell_{\text{hit}}, \ell_{\text{miss}}$) [S2, unofficial].
*   **When to use:** Robotics volumetric mapping with LiDAR data [S1].
*   **When not to use:** When non-occupancy properties (e.g., distance) are required [S2, unofficial].

## Background
The method was introduced in the wavemap framework for robotics volumetric mapping [S1]. It builds on OctoMap's log-odds integration approach [S2, unofficial]. The framework aims to provide accurate mapping of large, unstructured environments for autonomous robots [S1].

## Sources
[S1] Efficient volumetric mapping of multi-scale environments ..., https://www.roboticsproceedings.org/rss19/p065.pdf, fetched 2026-10-26
[S2] Dynamic GPU Sparse Grids and Voxel-sequential ..., https://research.nvidia.com/labs/prl/sifakis2026dynamic/dynamic_grids.pdf, fetched 2026-10-29 (unofficial)
