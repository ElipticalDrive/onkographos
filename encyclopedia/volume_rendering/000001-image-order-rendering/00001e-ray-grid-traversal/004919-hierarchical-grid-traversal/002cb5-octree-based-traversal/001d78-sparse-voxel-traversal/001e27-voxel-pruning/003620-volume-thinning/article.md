# Volume Thinning

Also called: Gradient-Curvature Sparseness, Iso-surface Dissolution

## Overview
Volume Thinning is a visibility reduction technique that progressively dissolves a volumetric dataset into a sparse wireframe-like structure by sorting voxels according to gradient and curvature magnitudes [S1, S2, unofficial]. It sets itself apart from sibling methods by explicitly targeting the geometric features of iso-surfaces, reducing the volume first to strong iso-surfaces and then dissolving those surfaces into high-curvature lines (ridges and valleys) [S1, S2, unofficial].

## How it works
**Inputs and Outputs**
- Input: A voxelized volume dataset $V$ with dimensions $D_x \times D_y \times D_z$.
- Input: A sparseness factor $\alpha \in [0, 1]$, where 0 represents the full volume and 1 represents the most sparse representation [S1, S2, unofficial].
- Output: A subset of voxels $V_{sparse} \subseteq V$ determined by $\alpha$, or a visibility mask for rendering [S1, S2, unofficial].

**Data Structures**
- A list of all active voxels in the volume.
- Each voxel entry stores:
  - Coordinates $(x, y, z)$
  - Scalar value $s$
  - Gradient magnitude $g = \|\nabla s\|$
  - Curvature magnitude $k$ of the iso-surface passing through the voxel [S1, S2, unofficial].

**Algorithm Steps**
1. **Compute Gradient Magnitude**: For each voxel, calculate the gradient magnitude $g$. This is typically the Euclidean norm of the gradient vector $\nabla s = (\frac{\partial s}{\partial x}, \frac{\partial s}{\partial y}, \frac{\partial s}{\partial z})$ [S1, S2, unofficial].
2. **Compute Curvature Magnitude**: Calculate the curvature magnitude $k$ of the iso-surface through the voxel. The exact formula for curvature is not specified in the sources, but it represents the local geometric complexity of the iso-surface [S1, S2, unofficial].
3. **Sort Voxels**: Sort all voxels based on two keys:
   - Primary key: Gradient magnitude $g$ (descending) [S1, S2, unofficial].
   - Secondary key: Curvature magnitude $k$ (descending) [S1, S2, unofficial].
4. **Apply Sparseness Factor**:
   - Determine the number of voxels to retain: $N_{retain} = \lfloor \alpha \cdot N_{total} \rfloor$, where $N_{total}$ is the total number of voxels [S1, S2, unofficial].
   - Select the top $N_{retain}$ voxels from the sorted list [S1, S2, unofficial].
   - The selection process implicitly follows the two-stage reduction:
     - First, voxels with high gradient magnitudes are retained, reducing the volume to strong iso-surfaces [S1, S2, unofficial].
     - As $\alpha$ decreases further, voxels with high curvature magnitudes are retained, dissolving iso-surfaces into high-curvature lines (ridges and valleys) [S1, S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volume_thinning(volume, sparseness_factor):
    """
    Performs volume thinning on a voxel dataset.
    
    Args:
        volume: A 3D array of scalar values.
        sparseness_factor: A float between 0 and 1. 0 means full volume, 1 means most sparse.
        
    Returns:
        A boolean mask of the same shape as volume, indicating which voxels are retained.
    """
    # Step 1: Compute gradient magnitude
    # Note: Exact gradient computation method (e.g., central differences) is not specified.
    # [G] Assuming standard central differences for interior voxels.
    gx, gy, gz = compute_gradients(volume)
    gradient_magnitude = np.sqrt(gx**2 + gy**2 + gz**2)
    
    # Step 2: Compute curvature magnitude
    # Note: Exact curvature formula is not specified in sources.
    # [G] Assuming a standard discrete curvature approximation based on gradient and Hessian.
    curvature_magnitude = compute_curvature(volume, gx, gy, gz)
    
    # Step 3: Sort voxels
    # Flatten the volume and associated metrics
    voxels = np.argwhere(volume) # Get coordinates of all voxels
    g_values = gradient_magnitude[voxels[:,0], voxels[:,1], voxels[:,2]]
    k_values = curvature_magnitude[voxels[:,0], voxels[:,1], voxels[:,2]]
    
    # Create an array of indices to sort by
    # Primary key: gradient magnitude (descending)
    # Secondary key: curvature magnitude (descending)
    sort_indices = np.lexsort((-k_values, -g_values))
    
    # Step 4: Apply sparseness factor
    total_voxels = len(voxels)
    num_retain = int(sparseness_factor * total_voxels)
    
    # Select top voxels based on sorted order
    selected_indices = sort_indices[:num_retain]
    
    # Create output mask
    mask = np.zeros_like(volume, dtype=bool)
    mask[voxels[selected_indices, 0], voxels[selected_indices, 1], voxels[selected_indices, 2]] = True
    
    return mask

def compute_gradients(volume):
    # [G] Standard central difference implementation
    gx = np.zeros_like(volume)
    gy = np.zeros_like(volume)
    gz = np.zeros_like(volume)
    
    gx[1:-1, :, :] = (volume[2:, :, :] - volume[:-2, :, :]) / 2.0
    gy[:, 1:-1, :] = (volume[:, 2:, :] - volume[:, :-2, :]) / 2.0
    gz[:, :, 1:-1] = (volume[:, :, 2:] - volume[:, :, :-2]) / 2.0
    
    # Handle boundaries with forward/backward differences
    gx[0, :, :] = volume[1, :, :] - volume[0, :, :]
    gx[-1, :, :] = volume[-1, :, :] - volume[-2, :, :]
    gy[:, 0, :] = volume[:, 1, :] - volume[:, 0, :]
    gy[:, -1, :] = volume[:, -1, :] - volume[:, -2, :]
    gz[:, :, 0] = volume[:, :, 1] - volume[:, :, 0]
    gz[:, :, -1] = volume[:, :, -1] - volume[:, :, -2]
    
    return gx, gy, gz

def compute_curvature(volume, gx, gy, gz):
    # [G] Placeholder for curvature computation.
    # The sources do not specify the exact formula for curvature magnitude.
    # Common approaches involve the trace of the Hessian or shape tensor.
    # This function needs to be implemented based on the specific curvature definition used in the original paper.
    raise NotImplementedError("Curvature computation formula not provided in sources.")
```

**Parameters**
- Sparseness factor $\alpha$: Controls the level of thinning. Typical values shown in sources are 0.75, 0.5, and 0.25 [S2, unofficial].
- Gradient and Curvature thresholds: Not explicitly defined; the sorting implicitly handles the thresholds based on $\alpha$ [S1, S2, unofficial].

**Edge Cases and Pitfalls**
- Boundary voxels: Gradient and curvature computations at boundaries require special handling (e.g., one-sided differences) [G].
- Degenerate input: Voxels with zero gradient (flat regions) will have zero gradient magnitude and may be sorted to the end, potentially being removed early [S1, S2, unofficial].
- Precision: Floating-point precision in gradient and curvature calculations can affect the sorting order, especially for voxels with similar values [G].

## Subtopics

### [LightGaussian Pruning Scoring](3213-lightgaussian-pruning-scoring.md)
Uses a different scoring mechanism based on Gaussian splatting parameters.

### [FA-based Pruning](3236-fa-based-pruning.md)
Relies on Factor Analysis for voxel importance.

### [Importance-Driven Volume Rendering](7632-importance-driven-volume-rendering.md)
The parent method that includes Volume Thinning as one of its techniques.

### [Nonuniform Compression on Grid Models](11898-nonuniform-compression-on-grid-models.md)
Focuses on compression rather than visibility reduction.

## Operating characteristics
- Time Complexity: Sorting voxels takes $O(N \log N)$, where $N$ is the number of voxels [S1, S2, unofficial]. Gradient and curvature computations are $O(N)$ [G].
- Memory Complexity: Requires storing gradient and curvature values for all voxels, $O(N)$ [S1, S2, unofficial].
- Measured Performance: Tested on datasets of resolution $512 \times 512 \times 87$ (Leopard Gecko) and $256 \times 256 \times 610$ (Monster Study) [S1, S2, unofficial].
- Scaling: Performance scales with the number of voxels. The sorting step becomes a bottleneck for very large volumes [S1, S2, unofficial].
- Strengths: Provides a visually intuitive reduction to iso-surfaces and then to feature lines, preserving important geometric structures [S1, S2, unofficial].
- Limitations: Requires pre-computation of gradient and curvature, which can be memory-intensive [S1, S2, unofficial]. The exact curvature formula is not specified, making implementation dependent on assumptions [G].

## Background
Volume Thinning was introduced in the paper "Importance-Driven Volume Rendering" by Viola et al. in 2004 [S1, S2, unofficial]. It is part of a broader framework for importance-driven rendering, which aims to optimize rendering by focusing computational resources on visually significant parts of the volume [S1, S2, unofficial]. The method was tested on medical and biological datasets, such as the Leopard Gecko and Monster Study datasets [S1, S2, unofficial].

## Sources
[S1] Importance-Driven Volume Rendering, https://www.researchgate.net/publication/220943935_Importance-Driven_Volume_Rendering, fetched 2026-11-03 (unofficial)
[S2] Importance-Driven Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2004/Viola-2004-ImpX/Viola-2004-ImpX-paper.pdf, fetched 2026-10-29 (unofficial)
