# Symmetric Voxel Grid
Also called: Symmetric arrangement, Symmetric feature arrangement

## Overview
The Symmetric Voxel Grid is a feature arrangement strategy for explicit voxel grid decoders that places features symmetrically with respect to the scene center to ensure continuity at region boundaries [S1, S2]. It solves the discontinuity problem inherent in linear feature configurations without the model size overhead of overlapping regions [S1, S2]. Unlike overlapping approaches which increase feature count by $3n^2 + 3n + 1$, this method maintains a compact grid while forcing neighboring decoders to see identical features at seams [S1, S2].

## How it works
**Inputs and Outputs**
- Input: A scene bounding volume divided into $n \times n \times n$ regions (or a grid of decoders).
- Output: A voxel grid where feature values are assigned to vertices/voxels such that symmetry constraints are met.

**Data Structures**
- The grid consists of discrete voxels. Each voxel vertex holds features accessed by adjacent decoders.
- Let $n$ be the number of subdivisions along one axis. The total number of regions is $n^3$.

**Steps**
1. **Define Regions**: Partition the scene space into a regular grid of $n \times n \times n$ regions. Each region $e_i$ has an associated decoder $\theta_i$.
2. **Symmetric Feature Assignment**: Assign features to voxel vertices such that any vertex $v$ on the boundary between two regions $e_1$ and $e_2$ presents the same feature value to both $\theta_1$ and $\theta_2$. This is achieved by mirroring feature values across the scene center.
   - In a linear configuration, a boundary vertex might have feature $f_a$ for region $e_1$ and $f_b$ for region $e_2$ where $f_a \neq f_b$, causing discontinuities [S1, S2].
   - In the symmetric configuration, the feature at $v$ is identical for all neighboring decoders [S1, S2].
3. **Decoder Initialization**: Initialize all density decoders $\theta_i$ with the same weights to further improve continuity during training [S1, S2].

**Formulas**
- Overhead of alternative overlapping method: The number of additional features required by overlapping regions in an explicit grid of size $n^3$ is:
  $$ \Delta N = 3n^2 + 3n + 1 $$
  This growth reduces the effectiveness of the explicit grid approach [S1, S2].

**Parameters**
- $n$: The number of subdivisions per axis. Typical values are not explicitly given in the sources, but the method scales with $n$.

**Edge Cases and Pitfalls**
- **Linear Configuration Discontinuity**: Using a simple repeating linear order for features introduces discontinuities at seams because different decoders see different features at the boundary [S1, S2].
- **Model Size Explosion**: Attempting to fix linear discontinuities by overlapping regions increases the model size significantly by $3n^2 + 3n + 1$ features [S1, S2].
- **Pruning Benefits**: The symmetric configuration increases the probability of features being in empty space for all regions, allowing more aggressive pruning and reducing the final stored feature count [S1, S2].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def assign_symmetric_features(grid, scene_center):
    """
    Assigns features to a voxel grid symmetrically with respect to the scene center.
    
    Args:
        grid: 3D array of shape (N, N, N) representing voxel vertices.
        scene_center: 3D vector representing the center of the scene.
        
    Returns:
        grid: Modified 3D array with symmetric feature assignment.
    """
    N = grid.shape[0]
    # Iterate through all voxel vertices
    for x in range(N):
        for y in range(N):
            for z in range(N):
                # Calculate position relative to center
                pos = grid[x, y, z].position - scene_center
                
                # Mirror position to find symmetric counterpart
                # The symmetric counterpart is at -pos relative to center
                # Map back to grid indices
                x_sym = N - 1 - x
                y_sym = N - 1 - y
                z_sym = N - 1 - z
                
                # Ensure symmetry: feature at (x,y,z) must equal feature at (x_sym, y_sym, z_sym)
                # If not already set, copy from symmetric counterpart
                if grid[x, y, z].feature is None:
                    if grid[x_sym, y_sym, z_sym].feature is not None:
                        grid[x, y, z].feature = grid[x_sym, y_sym, z_sym].feature
                    else:
                        # Initialize with default or shared weight
                        grid[x, y, z].feature = get_shared_feature()
                else:
                    # Ensure the counterpart matches
                    grid[x_sym, y_sym, z_sym].feature = grid[x, y, z].feature
                    
    return grid

def initialize_decoders(decoders):
    """
    Initializes all density decoders with the same weights.
    
    Args:
        decoders: List of decoder objects.
    """
    shared_weights = get_initial_weights()
    for decoder in decoders:
        decoder.weights = shared_weights
```

## Subtopics

### Linear Configuration
A naive arrangement where features are repeated in the same order for each region. It introduces discontinuities at seams because different decoders see different features at the boundary [S1, S2]. Choose only if model size is not a concern and discontinuities are tolerable. [Voxel Grid Decoders](2219-voxel-grid-decoders.md)

### Overlapping Regions
An alternative approach where regions overlap to ensure shared features at boundaries. It increases model size by $3n^2 + 3n + 1$ features [S1, S2]. Choose if symmetry is difficult to implement and extra memory is available. [Voxel Grid Decoders](2219-voxel-grid-decoders.md)

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: The symmetric arrangement does not increase the number of features needed for seamless reconstruction compared to a non-overlapping grid [S1, S2]. It reduces the number of features stored after pruning because it increases the probability of features being in empty space for all regions [S1, S2].
- **Measured Performance**: The method is described as "very effective to improve the model quality and reduce its size" [S1, S2]. Specific metrics (PSNR, FPS) are not provided in the notes.
- **Strengths**: Avoids discontinuities without increasing model size; enables more aggressive pruning; improves training continuity via shared decoder initialization.
- **Limitations**: Requires symmetric scene center definition; may not be optimal for highly asymmetric scenes where symmetry does not reflect natural feature distribution.
- **When to use**: When using explicit voxel grids with multiple decoders and seam discontinuities are a problem, and model size is a constraint.
- **When not to use**: When the scene is highly asymmetric and symmetry does not capture the underlying structure, or when overlapping regions are already acceptable.

## Background
The Symmetric Voxel Grid was introduced in the paper "NeRFLight: Fast and Light Neural Radiance Fields using a Shared Feature Grid" by Rivas-Manzaneque et al. (CVPR 2023) [S1, S2]. It addresses issues in explicit neural field representations where linear feature arrangements cause seams. The method builds on the concept of shared feature grids but specifically optimizes the spatial arrangement of those features to ensure continuity [S1, S2].

## Sources
[S1] NeRFLight: Fast and Light Neural Radiance Fields using a Shared Feature Grid, https://doi.org/10.1109/cvpr52729.2023.01195, fetched 2026-10-26
[S2] NeRFLight: Fast and Light Neural Radiance Fields using a Shared Feature Grid, https://openaccess.thecvf.com/content/CVPR2023/html/Rivas-Manzaneque_NeRFLight_Fast_and_Light_Neural_Radiance_Fields_Using_a_Shared_CVPR_2023_paper.html, fetched 2026-10-27
