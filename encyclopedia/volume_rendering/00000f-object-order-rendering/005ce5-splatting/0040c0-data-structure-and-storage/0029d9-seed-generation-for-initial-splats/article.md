# Seed Generation for Initial Splats

Also called: Gradient and Uniform Grid Seeding, Initial Splat Position Generation

## Overview
This method generates the initial set of splat primitives for volume rendering by combining gradient-based edge detection with uniform grid sampling [S1, unofficial]. It solves the problem of sparse or uneven primitive distribution, ensuring that high-frequency details are captured by gradient-weighted sampling while smooth regions are covered by a uniform grid [S1, unofficial]. It differs from sibling methods like [World-aligned lattice octree bake](11637-world-aligned-lattice-octree-bake.md) by using a fixed budget split (60/40) rather than hierarchical spatial partitioning, and from [Splat Data Structure (ISplatSource)](10289-splat-data-structure-isplatsource.md) by focusing on the *creation* of positions rather than their *storage* [S1, unofficial].

## How it works
**Inputs:**
- Volume data $V$ with dimensions $N_x \times N_y \times N_z$ (total voxels $N_{vox}$).
- Target seed count $N_{target}$ (default calculation below).
- Gradient threshold $\tau = 0.1$.
- Exclusion radius $r = 2$ voxels.
- Grid coverage share $S_{grid} = 0.4$ (40%).

**Outputs:**
- A list of initial splat positions $P = \{p_1, p_2, ..., p_k\}$ where $k \le N_{target}$.

**Data Structures:**
- `GradientMap`: 3D array of normalized gradient magnitudes, same size as $V$.
- `CandidateList`: List of voxel coordinates $(x,y,z)$ passing the gradient threshold.
- `PoissonDiskSampler`: Maintains a grid of exclusion zones to enforce minimum distance $r$.

**Steps:**
1. **Compute Gradients:** Calculate Sobel gradients in all three dimensions for every voxel. Compute the normalized gradient magnitude $g(x,y,z)$ for each voxel [S1, unofficial].
2. **Filter Candidates:** Identify voxels where $g(x,y,z) > \tau$. Store these in `CandidateList` [S1, unofficial].
3. **Poisson-Disk Sampling:** Apply gradient-weighted Poisson-disk sampling on `CandidateList` with exclusion radius $r=2$. This selects a subset of high-gradient voxels, ensuring they are spatially separated [S1, unofficial]. Let this set be $P_{edge}$.
4. **Calculate Grid Spacing:** Determine the number of grid seeds $N_{grid}$ required to fill the remaining budget ($40\%$ of $N_{target}$). The grid spacing $s$ is calculated such that the volume is tiled:
   $$ s = \max\left(2, \left\lfloor \left(\frac{N_{vox}}{N_{grid}}\right)^{1/D} \right\rfloor \right) $$
   where $D$ is the dimensionality (typically 3) [S1, unofficial]. Note: The source defines the spacing formula directly based on the ratio, ensuring the grid covers the volume with at least 2-voxel spacing [S1, unofficial].
5. **Generate Uniform Seeds:** Create a uniform grid with spacing $s$ across the volume bounds. Let this set be $P_{grid}$ [S1, unofficial].
6. **Combine and Cap:** Merge $P_{edge}$ and $P_{grid}$. If the total count exceeds $N_{target}$, the implementation may cull or prioritize (the source notes the retained count falls below the requested count after post-fit cull, implying the generation step aims for the target) [S1, unofficial].
7. **GPU Execution:** Run the seeding process on GPU (CUDA/MPS) if available [S1, unofficial].

**Parameters:**
- **Seed Target ($N_{target}$):**
  $$ N_{target} = \min\left(10000, \max\left(100, \left\lfloor \frac{N_{vox}}{2^{1/D}} \right\rfloor\right)\right) $$
  [S1, unofficial]
- **Gradient Threshold:** $0.1$ [S1, unofficial].
- **Exclusion Radius:** $2$ voxels [S1, unofficial].
- **Grid Share:** $40\%$ of the seed budget [S1, unofficial].

**Edge Cases:**
- If $N_{vox}$ is small, $N_{target}$ defaults to $100$ [S1, unofficial].
- Grid spacing is clamped to a minimum of $2$ voxels to prevent overly dense uniform sampling in small volumes [S1, unofficial].
- The final retained count may be lower than $N_{target}$ due to post-fit culling (see [Dynamic operations](link)) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_seeds(volume, N_vox, D=3):
    # 1. Compute normalized gradient magnitude
    gradients = compute_sobel_gradients(volume)
    norm_grads = normalize_gradient_magnitude(gradients)
    
    # 2. Filter candidates
    candidates = []
    for voxel in volume.voxels:
        if norm_grads[voxel] > 0.1:
            candidates.append(voxel)
            
    # 3. Poisson-disk sampling for edges (60% budget)
    # Note: Exact Poisson-disk implementation details are not provided in source [G]
    # Assuming a standard implementation that respects exclusion radius
    edge_seeds = poisson_disk_sample(candidates, exclusion_radius=2)
    
    # 4. Calculate grid parameters
    # Target total seeds
    raw_target = max(100, floor(N_vox / (2 ** (1/D))))
    N_target = min(10000, raw_target)
    
    # Grid gets 40% of the budget
    N_grid_seeds = floor(0.4 * N_target)
    
    # Calculate spacing s such that grid tiles the volume
    # Formula from source: (N_vox / N_grid)^(1/D)
    # We need N_grid such that N_grid * (1/s^D) approximates coverage?
    # Source says: "Grid spacing set so that its 40% share ... tiles the volume"
    # and gives formula for spacing: (N_vox / N_grid)^(1/D)
    # This implies N_grid is the count of grid cells? Or is N_grid the spacing?
    # Re-reading: "Grid spacing ... ((N_vox / N_grid)^(1/D) ...)"
    # It seems N_grid in the formula refers to the number of grid points?
    # Let's assume N_grid_count is the number of uniform seeds to generate.
    
    # If N_grid_seeds is 0, skip grid
    if N_grid_seeds == 0:
        return edge_seeds
        
    # Calculate spacing
    # The source formula is ambiguous on whether N_grid is count or spacing.
    # "Grid spacing set so that its 40% share ... tiles the volume"
    # Usually, spacing s = Volume_Dimension / Grid_Count_Dimension.
    # The formula (N_vox / N_grid)^(1/D) suggests N_grid is the total number of grid seeds.
    spacing = max(2, floor((N_vox / N_grid_seeds) ** (1/D)))
    
    # 5. Generate uniform grid
    grid_seeds = []
    for x in range(0, volume.width, spacing):
        for y in range(0, volume.height, spacing):
            for z in range(0, volume.depth, spacing):
                grid_seeds.append(Voxel(x, y, z))
                
    # 6. Combine
    all_seeds = edge_seeds + grid_seeds
    
    # 7. Cap at N_target (if over)
    if len(all_seeds) > N_target:
        all_seeds = all_seeds[:N_target]
        
    return all_seeds
```

## Subtopics

### Scale-Hierarchical Decomposition Seeder
A seeder that decomposes the volume across scales to determine seed positions [S1, unofficial]. It is combinable with the default seeder and is particularly useful for multi-scale data structures [S1, unofficial].

### Intensity-Weighted Peaks Seeder
A seeder that selects positions based on intensity-weighted peaks, specifically used on sparse residuals in progressive fitting scenarios [S1, unofficial]. It complements the gradient-based approach by targeting high-intensity features that may not have strong gradients [S1, unofficial].

### Seed Selection Preview
A technique that provides a visual preview of the selected seeds during the seed selecting process, allowing for user interaction or verification [S2, unofficial]. This is part of an enhanced isosurface rendering approach involving voxel selecting and selection-based coloring [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on gradient computation ($O(N_{vox})$) and Poisson-disk sampling (typically $O(N_{seeds})$ with spatial hashing) [S1, unofficial].
- **Memory Complexity:** $O(N_{vox})$ for gradient storage and candidate lists [S1, unofficial].
- **Performance:** The single-pass implementation has a wall-time ratio of 0.70–1.75× compared to two-pass, with a median of 1.07× [S1, unofficial].
- **Strengths:** Ensures coverage of smooth regions (via grid) and edges (via gradient) [S1, unofficial].
- **Limitations:** The final count may be lower than requested due to post-fit culling [S1, unofficial].

## Background
The method is described in the context of "Luxar," a Gaussian splatting framework for microscopy and scalable interactive web visualization of multidimensional scientific data [S1, unofficial]. The specific seed generation strategy (60% gradient, 40% uniform) is the default "auto" seeder used in single-pass fitting reported in the source [S1, unofficial]. The concept of seed preview is associated with enhanced isosurface rendering techniques [S2, unofficial].

## Sources
[S1] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
[S2] Efficient and Effective Volume Visualization with Enhanced Isosurface Rendering, https://doi.org/10.48550/arxiv.1202.5360, fetched 2026-10-26 (unofficial)
