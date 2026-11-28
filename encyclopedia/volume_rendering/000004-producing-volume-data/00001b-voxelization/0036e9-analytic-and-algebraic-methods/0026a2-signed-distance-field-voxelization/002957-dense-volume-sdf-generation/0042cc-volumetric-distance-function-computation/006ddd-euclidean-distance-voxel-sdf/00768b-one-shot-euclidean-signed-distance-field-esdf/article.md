# One-shot Euclidean Signed Distance Field (ESDF)

Also called: Fixed-point monotone-min Jacobi sweep ESDF, One-shot ESDF

## Overview
The One-shot Euclidean Signed Distance Field (ESDF) method computes a proper Euclidean signed distance field from a Truncated Signed Distance Field (TSDF) grid using a cold-start approach [S1, unofficial]. It involves topology dilation to allocate a support band around the zero-crossing, seeding the boundary, and performing a fixed-point monotone-min Jacobi sweep over 26-connectivity neighbors [S1, unofficial]. It sets itself apart from incremental variants by always recomputing the entire field from scratch, ensuring global consistency at the cost of higher per-frame latency [S1, unofficial].

## How it works
**Inputs:**
- TSDF grid $G_{tsdf}$ containing distance values.
- Sidecars for TSDF (e.g., occupancy or weight, denoted $W_{tsdf}$ and $O_{tsdf}$ in source notation, though only distance is strictly required for the sweep logic described).
- Truncation distance $\tau_{tsdf}$.
- Maximum ESDF distance $\tau_{esdf}$ (clamping bound).

**Outputs:**
- ESDF grid $G_{esdf}$ containing exact Euclidean signed distances.
- Signed-distance sidecar (often the same grid or a derived mask).

**Data Structures:**
- $G_{esdf}$: A sparse or dense grid of floating-point values. Initialized to $+\infty$ or a large sentinel value before the sweep.
- $N_{26}(v)$: The set of 26 neighbors of voxel $v$ in a 3D grid (all adjacent voxels sharing a face, edge, or corner).
- $x_v, x_u$: The 3D coordinates of the centers of voxels $v$ and $u$.

**Parameters:**
- $\tau_{tsdf}$: TSDF truncation distance. Typical value: $3 \times$ voxel size [S1, unofficial].
- $\tau_{esdf}$: Maximum distance to compute. Values beyond this are clamped [S1, unofficial].

**Steps:**
1. **Topology Dilation:** Expand the active region of the grid to include a band of voxels around the TSDF zero-crossing. The dilation radius is $\lceil \tau_{esdf} / \text{voxel\_size} \rceil$. This ensures that voxels potentially within the maximum ESDF distance of the surface are included in the computation [S1, unofficial].
2. **Seeding:** Initialize $G_{esdf}$ using a seed kernel. For each voxel $v$ in the dilated grid:
   - If $G_{tsdf}[v] > 0$ and $|G_{tsdf}[v]| < \tau_{tsdf}$, set $G_{esdf}[v] = \text{sign}(G_{tsdf}[v]) \cdot |G_{tsdf}[v]|$.
   - Otherwise, set $G_{esdf}[v] = +\infty$.
   - Note: The source formula uses $\text{sign}(G_{tsdf}[u]) \cdot |G_{tsdf}[u]|$ inside the min, implying the seed sets the initial boundary conditions based on the TSDF value [S1, unofficial].
3. **Fixed-Point Monotone-Min Jacobi Sweep:** Iterate until convergence:
   - Initialize a convergence flag $changed \leftarrow \text{True}$.
   - While $changed$ is true:
     - Set $changed \leftarrow \text{False}$.
     - For each voxel $v \in G_{esdf}$ (iterated in parallel, e.g., via Voxel-Block-Mapping):
       - Compute the candidate distance from all 26 neighbors:
         $$ G_{new} \leftarrow \min_{u \in N_{26}(v)} \left( G_{esdf}[u] + \text{sign}(G_{tsdf}[u]) \cdot \|x_v - x_u\| \right) $$
       - If $|G_{new}| < |G_{esdf}[v]|$:
         - Update $G_{esdf}[v] \leftarrow G_{new}$.
         - Set $changed \leftarrow \text{True}$.
4. **Clamping:** Clamp the final values in $G_{esdf}$ to the range $[-\tau_{esdf}, +\tau_{esdf}]$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def one_shot_esdf(G_tsdf, tau_tsdf, tau_esdf, voxel_size):
    # 1. Topology Dilation
    # Determine the set of voxels to process based on TSDF occupancy/distance
    # This step allocates the support band. Let G_esdf be the grid for these voxels.
    # Initialize G_esdf with infinity
    G_esdf = {v: float('inf') for v in dilated_voxels(G_tsdf, ceil(tau_esdf / voxel_size))}
    
    # 2. Seeding
    for v in G_esdf:
        tsdf_val = G_tsdf[v]
        if tsdf_val > 0 and abs(tsdf_val) < tau_tsdf:
            G_esdf[v] = sign(tsdf_val) * abs(tsdf_val)
        else:
            G_esdf[v] = float('inf')
            
    # 3. Fixed-Point Sweep
    changed = True
    while changed:
        changed = False
        # Parallel iteration over voxels
        for v in G_esdf:
            min_val = float('inf')
            for u in neighbors_26(v):
                # Access neighbor value; if u is not in G_esdf, treat as inf
                val_u = G_esdf.get(u, float('inf'))
                tsdf_u = G_tsdf.get(u, 0) # Or handle out-of-bounds appropriately
                
                # Calculate Euclidean distance
                dist = distance(v.center, u.center)
                
                # Candidate value
                candidate = val_u + sign(tsdf_u) * dist
                if candidate < min_val:
                    min_val = candidate
            
            # Update if improved
            if abs(min_val) < abs(G_esdf[v]):
                G_esdf[v] = min_val
                changed = True
                
    # 4. Clamping
    for v in G_esdf:
        G_esdf[v] = clamp(G_esdf[v], -tau_esdf, tau_esdf)
        
    return G_esdf
```

**Edge Cases and Pitfalls:**
- **Precision:** The monotone-min property ensures convergence, but floating-point precision errors can cause oscillations if not handled. The condition $|G_{new}| < |G_{esdf}[v]|$ helps prevent infinite loops from minor precision fluctuations [S1, unofficial].
- **Boundaries:** Voxels outside the dilated grid are implicitly $+\infty$. The sign of the TSDF neighbor $u$ is critical; if $u$ is outside the TSDF domain, its sign must be defined (typically 0 or based on occupancy) [S1, unofficial].
- **Degenerate Input:** If the TSDF is empty or fully truncated, the ESDF may remain infinite or zero depending on seeding logic [S1, unofficial].

## Subtopics

### Incremental ESDF with dirty-mask short-circuit
An incremental warm-start variant that reuses the previous ESDF grid to save computation [S1, unofficial]. It uses a "dirty mask" to identify only the voxels that have changed in the TSDF. If the dirty mask is empty, it returns the previous ESDF immediately. Otherwise, it seeds only the dirty voxels and runs the fixed-point sweep only on affected areas or the whole grid with warm-started values [S1, unofficial]. This is chosen when the scene changes incrementally and low latency is required, whereas the one-shot method is used for full consistency or large changes.

## Operating characteristics
- **Time Complexity:** The sweep converges in $O(D)$ iterations where $D$ is the diameter of the support band in voxels, with $O(N)$ work per iteration where $N$ is the number of voxels in the dilated grid [S1, unofficial].
- **Memory Complexity:** Proportional to the number of voxels in the dilated grid, including sidecars [S1, unofficial].
- **Performance:** The VBM (Voxel-Block-Mapping) parallel iteration provides significant speedup over per-leaf-slot dispatch, especially for low-occupancy stencils [S1, unofficial].
- **Strengths:** Guarantees global Euclidean distance accuracy; simple parallel implementation.
- **Limitations:** High latency for large scenes due to full recomputation; memory intensive for large dilated grids.
- **When to use:** When global consistency is required, or when the scene undergoes large changes. Not ideal for real-time incremental updates where only small parts of the map change.

## Background
The method is part of the family of Euclidean Distance Voxel SDF generation techniques. It builds on the standard parallel-chamfer envelope for 26-connectivity used by systems like FIESTA [Han et al. 2019] and nvblox [Millane et al. 2024] [S1, unofficial]. The specific "one-shot" cold-start algorithm is detailed in the context of dynamic GPU sparse grids, likely from NVIDIA Research (Sifakis et al.) [S1, unofficial].

## Sources
[S1] Dynamic GPU Sparse Grids and Voxel-sequential ..., https://research.nvidia.com/labs/prl/sifakis2026dynamic/dynamic_grids.pdf, fetched 2026-10-29 (unofficial)
