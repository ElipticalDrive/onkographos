# SIRT with subsets and constraints

Also called: Constrained SIRT with subsets, SIRT with subsets

## Overview
This method enhances the Simultaneous Iterative Reconstruction Technique (SIRT) by partitioning projection data into subsets to manage memory usage and applying hard bounds to voxel values to enforce physical constraints. It solves the problem of reconstructing volumes from large datasets that exceed available RAM while preventing non-physical values (e.g., negative attenuation) [S1, unofficial]. It differs from standard SIRT by processing data in chunks rather than all at once, and from SART by retaining SIRT's simultaneous update nature within each subset iteration rather than updating voxel-by-voxel per ray [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:**
  - `projections`: 3D array of shape `[num_views, height, width]` (dtype `float32`) [S1, unofficial].
  - `geometry`: Geometry description object (types: `'simple'`, `'static_offsets'`, `'linear_offsets'`) [S1, unofficial].
  - `settings`: Configuration object containing `subsets`, `sorting`, `bounds`, `update_residual`, and `iterations` [S1, unofficial].
- **Outputs:**
  - `volume`: 3D array of reconstructed voxel values (dtype `float32`) [S1, unofficial].

**Data Structures:**
- **Settings Object:**
  - `subsets` (int): Number of projection partitions. Default 10 [S1, unofficial].
  - `sorting` (str): Order of subsets. Options: `'sequential'` or `'equidistant'` [S1, unofficial].
  - `bounds` (list): `[lower, upper]` clamping values for voxel intensities [S1, unofficial].
  - `update_residual` (bool): Whether to compute and store residual norms [S1, unofficial].
  - `iterations` (int): Total number of reconstruction passes [S1, unofficial].

**Algorithm Steps:**
1. **Initialization:** Create a zero-initialized volume array `vol_rec` with shape matching the target reconstruction grid (typically `[num_views, height, width]` or derived from geometry) [S1, unofficial].
2. **Subset Partitioning:** Divide the `projections` array into `N` subsets based on `settings.subsets`. If `settings.sorting` is `'equidistant'`, distribute views uniformly across angular space; if `'sequential'`, split the array contiguously [S1, unofficial].
3. **Iteration Loop:** For each iteration `k` from 1 to `settings.iterations`:
    a. **Subset Processing:** Iterate through each subset `s` of the partitioned projections.
    b. **Forward/Backward Projection:** Perform one SIRT update step using only the projections in subset `s`. This involves forward projecting the current volume estimate to generate simulated projections, computing the residual (difference between measured and simulated projections), and backprojecting the residual to update the volume [S1, unofficial].
    c. **Normalization:** Apply SIRT normalization factors (ray length and voxel sensitivity) as defined in the parent SIRT method [S1, unofficial].
    d. **Residual Update (Optional):** If `settings.update_residual` is True, compute the L2 norm of the residual for the current subset and store it [S1, unofficial].
4. **Constraint Application:** After each full iteration (or optionally after each subset, depending on implementation specifics), apply bounds to the volume: `vol_rec = clip(vol_rec, min=settings.bounds[0], max=settings.bounds[1])` [S1, unofficial].
5. **Termination:** Return `vol_rec` after `settings.iterations` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def SIRT_with_subsets_and_constraints(projections, geometry, settings):
    """
    SIRT reconstruction with subsets and box constraints.
    
    Args:
        projections: numpy array of shape [num_views, height, width], dtype float32
        geometry: Geometry object
        settings: Object with attributes:
            - subsets: int (default 10)
            - sorting: str ('sequential' or 'equidistant')
            - bounds: list [lower, upper]
            - iterations: int
            - update_residual: bool
    """
    # Initialize volume
    # Note: Shape derivation from geometry is implementation-specific [G]
    num_views, height, width = projections.shape
    vol = numpy.zeros((num_views, height, width), dtype='float32')
    
    # Partition projections into subsets
    if settings.sorting == 'equidistant':
        # Distribute views uniformly
        indices = numpy.linspace(0, num_views - 1, settings.subsets, dtype=int)
        # Create subsets by grouping views
        subsets = []
        for i in range(settings.subsets):
            start = indices[i]
            end = indices[(i + 1) % settings.subsets]
            if start < end:
                subsets.append(projections[start:end])
            else:
                subsets.append(numpy.concatenate([projections[start:], projections[:end]]))
    else: # sequential
        step = num_views // settings.subsets
        subsets = numpy.array_split(projections, settings.subsets, axis=0)
    
    rnorms = []
    
    for iteration in range(settings.iterations):
        for subset_proj in subsets:
            # Perform one SIRT update step with the current subset
            # This involves:
            # 1. Forward project 'vol' using 'subset_proj' geometry to get simulated projections
            # 2. Compute residual: residual = subset_proj - simulated_projections
            # 3. Backproject 'residual' to update 'vol'
            # 4. Apply SIRT normalization factors (ray length, voxel sensitivity)
            # Note: The exact normalization formulas are defined in the parent SIRT article [S1, unofficial]
            vol = SIRT_update_step(vol, subset_proj, geometry)
            
            # Apply bounds if specified
            if settings.bounds:
                vol = numpy.clip(vol, a_min=settings.bounds[0], a_max=settings.bounds[1])
            
            # Update residual norm if requested
            if settings.update_residual:
                # Compute L2 norm of residual for this subset
                # Note: Residual computation requires forward projection of current vol [G]
                simulated = forward_project(vol, geometry)
                residual = subset_proj - simulated
                norm = numpy.linalg.norm(residual)
                rnorms.append(norm)
    
    return vol
```

**Parameters:**
- `iterations`: Number of reconstruction passes. Typical value: 20 [S1, unofficial].
- `subsets`: Number of projection partitions. Default: 10 [S1, unofficial].
- `bounds`: List `[lower, upper]` for clamping. Typical lower bound: 0 (non-negativity) [S1, unofficial].
- `sorting`: String, `'equidistant'` for uniform angular distribution [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Memory Overflow:** Large datasets may exceed RAM. Using subsets splits the data to fit in memory [S1, unofficial].
- **Precision:** Use `float32` for projections and volume to balance precision and memory [S1, unofficial].
- **Boundary Conditions:** Ensure geometry offsets are correctly applied to avoid artifacts at volume boundaries [S1, unofficial].

## Subtopics

### [SIRT with Local Box Constraints](19207-sirt-with-local-box-constraints.md)
Applies local box constraints derived from segmentation of a static reference scan to eliminate noise in specific voxel types (e.g., rock vs. fluid). Choose when material phases are known and global bounds are insufficient.

## Operating characteristics
- **Time Complexity:** $O(K \cdot N \cdot V)$, where $K$ is iterations, $N$ is subsets, and $V$ is volume size. Each subset requires a forward and backward projection [S1, unofficial].
- **Memory Complexity:** $O(V + P/N)$, where $P$ is total projection data size. Subsets reduce memory from $O(P)$ to $O(P/N)$ [S1, unofficial].
- **Performance:** Suitable for large datasets that do not fit in RAM. Convergence may be slower than full SIRT due to subset ordering, but memory efficiency is significantly improved [S1, unofficial].
- **Strengths:** Memory efficient, handles large datasets, enforces physical constraints [S1, unofficial].
- **Limitations:** Slower convergence than full SIRT, requires careful tuning of subset size and sorting [S1, unofficial].
- **When to Use:** Large CT datasets, memory-constrained environments, when physical bounds are known [S1, unofficial].
- **When Not to Use:** Small datasets where full SIRT fits in memory, when maximum convergence speed is critical [S1, unofficial].

## Background
Originated from the SIRT method, enhanced with subsets and constraints to address memory and physical constraint issues in tomographic reconstruction [S1, unofficial]. Implemented in the flexTOMO library [S1, unofficial].

## Sources
[S1] cicwi/flexTOMO, https://github.com/cicwi/flexTOMO, fetched 2026-10-27 (unofficial)
