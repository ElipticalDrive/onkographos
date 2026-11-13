# RGVis Region Growing

Also called: RGVis, Region Growing Based Visualization

## Overview
RGVis Region Growing is an interactive volume visualization technique that uses 3D region growing to assist users in locating features of interest and defining transfer functions or boundary surfaces [S1, unofficial]. It solves the problem of iterative trial-and-error in parameter selection by deriving a transfer function from the statistical properties of a partially grown region, or a complete boundary surface from a full region growing result [S1, unofficial]. It sets itself apart from its sibling [Adaptive Region Growing Cost Functions](14344-adaptive-region-growing-cost-functions.md) by providing a complete interactive workflow for seed selection, adaptive growth, and subsequent visualization aid (transfer function or surface), whereas the parent article defines the underlying cost functions. It differs from dual-domain interactive methods [Kniss02] by automatically suggesting a transfer function based on region statistics rather than requiring manual editing of histogram points [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** 3D volume data (scalar values), user interaction (seed point or line segment on a slice).
- **Output:** A set of connected voxels (region), a 2D transfer function (from partial growing), or a polygonal boundary surface (from full growing) [S1, unofficial].

**Data Structures:**
- **Volume Data:** 3D array of scalar values $v$. Gradient magnitudes $g$ are computed or available.
- **Region Set:** A collection of voxel coordinates $(x, y, z)$ belonging to the current region.
- **Queue/Stack:** Used for iterative or recursive neighbor expansion.
- **Statistics:** Running mean and standard deviation for data values ($\sigma_v$) and gradient magnitudes ($\sigma_g$) of voxels in the current region.

**Parameters:**
- $k$: A user-specified constant controlling the tolerance in the cost function $f_{ca}$ [S1, unofficial].
- Termination criterion: Either a fixed number of voxels (for partial growing) or exhaustion of valid neighbors (for full growing) [S1, unofficial].

**Steps:**

1. **Seed Selection:**
   - The user slices through the volume and selects a region of interest.
   - **Point Seed:** The user clicks a point. The system highlights the selected voxel and its 26 neighbors on a 2D scatterplot (scalar value vs. gradient magnitude). A blue rectangle indicates the standard deviation of the neighbors' values [S1, unofficial].
   - **Line Seed:** The user draws a line segment of 3D points. The interface displays preclassified zones for these points. The user selects a zone to determine the seed [S1, unofficial].

2. **Initialization:**
   - Select a starting voxel in the proximity of the seed. The choice is based on distance from the seed or the statistical properties of the neighborhood (e.g., homogeneity) [S1, unofficial].
   - Initialize the region set with this voxel.
   - Compute initial statistics ($\sigma_v, \sigma_g$) for the neighbors of the starting voxel [S1, unofficial].

3. **Region Growing Expansion:**
   - For each voxel in the region, visit its 26 immediate neighbors.
   - For each neighbor, evaluate the cost functions.
   - **Cost Function $f_{ca}$:**
     $$ f_{ca} = \frac{|v - v_s|}{k \sigma_v} $$
     where $v$ is the data value of the current neighbor, $v_s$ is the data value of the seed voxel, $k$ is the user constant, and $\sigma_v$ is the standard deviation of the data values of the neighbors [S1, unofficial].
   - **Adaptive Criterion $f_{cc}$:**
     The criterion is made adaptive by continuously recomputing $\sigma_v$, $\sigma_g$, and a parameter $p$ using the voxels collected so far in the region [S1, unofficial].
     *Note: The exact formula for $f_{cc}$ and $f_{cb}$ is not fully specified in the provided text, but the text states a voxel is included if the value returned by the cost function is less than 1 [S1, unofficial].*
   - If the cost function value is less than 1, add the neighbor to the region and the queue/stack for further expansion [S1, unofficial].

4. **Termination:**
   - **Partial Growing:** Stop when a predefined number of voxels (e.g., 5,000) is reached or after a short time limit (e.g., 0.03 seconds) [S1, unofficial].
   - **Full Growing:** Continue until all reachable neighbors satisfy the termination criterion or all voxels in the volume are examined [S1, unofficial].

5. **Post-Processing:**
   - **Transfer Function Generation:** Use the statistical properties (mean, standard deviation) of the voxels in the partial region to derive a 2D transfer function (scalar vs. gradient magnitude) [S1, unofficial].
   - **Boundary Surface Generation:** Use the voxels from the full region to construct a boundary surface (e.g., using marching cubes or similar surface reconstruction) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rgvis_region_growing(volume, seed_point, k, max_voxels=None, time_limit=None):
    """
    Performs region growing based on RGVis criteria.
    
    Args:
        volume: 3D array of scalar values.
        seed_point: (x, y, z) coordinates of the seed.
        k: User-specified constant for cost function.
        max_voxels: Optional limit for partial growing.
        time_limit: Optional time limit for partial growing.
        
    Returns:
        region: Set of (x, y, z) coordinates in the region.
        stats: Dictionary of statistical properties (mean_v, std_v, mean_g, std_g).
    """
    # Initialize region and queue
    region = set()
    queue = []
    
    # Select initial voxel based on proximity to seed or neighborhood stats
    # [G] Placeholder for specific selection logic not detailed in source
    start_voxel = select_initial_voxel(volume, seed_point)
    
    region.add(start_voxel)
    queue.append(start_voxel)
    
    # Compute initial statistics for neighbors of start_voxel
    # [G] Placeholder for neighbor selection and stat computation
    neighbors = get_26_neighbors(start_voxel, volume.shape)
    stats = compute_initial_stats(volume, neighbors)
    
    # Adaptive parameters
    sigma_v = stats['std_v']
    sigma_g = stats['std_g']
    p = stats['p'] # Parameter for adaptive criterion
    v_s = volume[seed_point] # Seed value
    
    while queue:
        current_voxel = queue.pop(0)
        
        # Check termination for partial growing
        if max_voxels and len(region) >= max_voxels:
            break
        if time_limit and time_elapsed() > time_limit:
            break
            
        current_neighbors = get_26_neighbors(current_voxel, volume.shape)
        
        for neighbor in current_neighbors:
            if neighbor in region:
                continue
                
            v_n = volume[neighbor]
            g_n = gradient_magnitude(neighbor, volume) # [G] Gradient computation assumed
            
            # Evaluate cost function f_ca
            # f_ca = |v_n - v_s| / (k * sigma_v)
            if sigma_v == 0:
                f_ca = 0 # [G] Avoid division by zero; assume homogeneous if sigma is 0
            else:
                f_ca = abs(v_n - v_s) / (k * sigma_v)
            
            # Evaluate adaptive criterion f_cc
            # [G] Exact formula for f_cc not provided in source. 
            # Assuming it uses sigma_v, sigma_g, p. 
            # Source states: "voxel is included if the value returned by the cost function is less than 1"
            # We assume f_cc is another cost function evaluated similarly.
            # For implementation, we check f_ca < 1 as a primary criterion from source.
            # The source mentions f_cc is adaptive and uses continuously recomputed stats.
            
            if f_ca < 1: # Primary criterion from source
                region.add(neighbor)
                queue.append(neighbor)
                
                # Update statistics for adaptive criterion
                # [G] Source says sigma_v, sigma_g, p are continuously recomputed using voxels collected so far.
                # This implies recomputing stats over the entire 'region' set or incrementally.
                # Incremental update is more efficient.
                update_stats_incremental(stats, v_n, g_n)
                sigma_v = stats['std_v']
                sigma_g = stats['std_g']
                p = stats['p']
                
    return region, stats

def compute_initial_stats(volume, neighbors):
    """Computes initial standard deviations for data values and gradient magnitudes."""
    values = [volume[n] for n in neighbors]
    gradients = [gradient_magnitude(n, volume) for n in neighbors]
    
    mean_v = sum(values) / len(values)
    std_v = sqrt(sum((v - mean_v)**2 for v in values) / len(values))
    
    mean_g = sum(gradients) / len(gradients)
    std_g = sqrt(sum((g - mean_g)**2 for g in gradients) / len(gradients))
    
    # [G] Parameter p is mentioned but its definition is not in the source.
    # Assuming p is a placeholder or derived from mean_g or similar.
    p = mean_g 
    
    return {'mean_v': mean_v, 'std_v': std_v, 'mean_g': mean_g, 'std_g': std_g, 'p': p}

def update_stats_incremental(stats, v, g):
    """Updates running statistics. Note: Standard deviation requires full recomputation or complex Welford's algorithm."""
    # [G] Simple incremental mean update. Std dev update is non-trivial.
    # For correctness, full recomputation over 'region' might be intended by source, 
    # but that is expensive. Welford's algorithm is the standard efficient way.
    # Here we assume a placeholder for the update logic.
    pass

def get_26_neighbors(voxel, shape):
    """Returns list of 26 neighbors within bounds."""
    x, y, z = voxel
    neighbors = []
    for dx in [-1, 0, 1]:
        for dy in [-1, 0, 1]:
            for dz in [-1, 0, 1]:
                if dx == 0 and dy == 0 and dz == 0:
                    continue
                nx, ny, nz = x + dx, y + dy, z + dz
                if 0 <= nx < shape[0] and 0 <= ny < shape[1] and 0 <= nz < shape[2]:
                    neighbors.append((nx, ny, nz))
    return neighbors

def gradient_magnitude(voxel, volume):
    """Computes gradient magnitude using central differences."""
    x, y, z = voxel
    # [G] Boundary handling assumed via padding or clamping
    gx = volume[x+1, y, z] - volume[x-1, y, z]
    gy = volume[x, y+1, z] - volume[x, y-1, z]
    gz = volume[x, y, z+1] - volume[x, y, z-1]
    return sqrt(gx**2 + gy**2 + gz**2)
```

## Subtopics

### [Partial Region Growing for Transfer Function Generation](14373-partial-region-growing-for-transfer-function-generation.md)
Partial Region Growing for Transfer Function Generation is a method to construct a 2D transfer function (classifier) for volume visualization by performing region growing on only a subset of the volume or a small subset of regions. It solves the problem of high computational cost associated with full region growing during the exploration stage of volume visualization. It sets itself apart from its parent method, RGVis Region Growing, by limiting the growth extent to achieve interactive speeds while still capturing enough statistical information to define a robust classifier for the overall features.

## Operating characteristics
- **Time Complexity:** Partial region growing is designed for interactivity. On a confocal microscopy ganglion data set (64×300×400 voxels), partial growing took 0.03 seconds to extract 5,000 voxels [S1, unofficial]. Full region growing is more expensive as it may examine all voxels in the volume.
- **Memory Complexity:** Proportional to the number of voxels in the region and the volume size for neighbor lookups.
- **Strengths:**
  - Interactive speed for feature exploration.
  - Automatically derives transfer functions, reducing manual effort compared to dual-domain methods [S1, unofficial].
  - Boundary surfaces from full growing represent features more completely than isosurfaces, capturing fine parts [S1, unofficial].
- **Limitations:**
  - Requires user interaction for seed selection.
  - Performance of full growing depends on volume size and feature connectivity.
  - The exact formula for the adaptive criterion $f_{cc}$ is not fully detailed, potentially making implementation ambiguous [S1, unofficial].
- **When to use:** When exploring volume data to locate features and define transfer functions quickly, or when generating boundary surfaces for detailed feature representation.
- **When not to use:** When fully automatic segmentation without user interaction is required, or when real-time performance on very large volumes is critical and partial growing is insufficient.

## Background
RGVis (Region Growing Based Visualization) was developed at UC Davis by Huang and Ma [S1, unofficial]. It is an interactive volume visualization technique that employs 3D region growing to assist in data exploration and visualization. The method was applied to confocal microscopy ganglion data sets and medical applications [S1, unofficial]. It builds on the concept of region growing for data segmentation, where the goal is to map input volume data into sets of connected voxels based on local properties [S1, unofficial]. The system supplements region growing with visual assessment methods and boundary surface modeling [S1, unofficial].

## Sources
[S1] RGVis: Region Growing Based Visualization Techniques for Volume Visualization, https://escholarship.org/uc/item/3td71868, fetched 2026-11-03 (unofficial)
