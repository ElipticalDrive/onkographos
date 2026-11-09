# Box-filtering Density Estimation


## Overview
Box-filtering density estimation transforms a binary slicemap into a density representation by filtering and downsampling the grid, where each resulting voxel stores the count of filled voxels from the original high-resolution grid [S1, unofficial]. It sets itself apart from sibling methods like Kernel Density Estimation by using exact integer arithmetic on binary slicemaps rather than probabilistic or smoothed field approximations, prioritizing computational speed over smoothness [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A binary slicemap volume $v$, where $v(x,y,z) \in \{0, 1\}$ represents occupancy.
- Output: A downsampled density volume $d$, where $d(x,y,z)$ is an integer count.

**Data Structures:**
- The slicemap $v$ is stored as a 3D array of binary values (e.g., bits or bytes).
- The density map $d$ is stored as a 3D array of integers. The resolution of $d$ is half that of $v$ in each dimension (for a 2x2x2 kernel).

**Method:**
The method constructs the density map by applying a box filter (summing neighbors) and downsampling. This process is dyadic, meaning any power-of-two kernel size can be achieved by repeated application of the 2x2x2 filter [S1, unofficial].

1.  **Filtering and Downsampling:** For each voxel $(i, j, k)$ in the output density grid, compute the sum of the $2 \times 2 \times 2$ block of voxels from the input slicemap that maps to this location.
2.  **Formula:** The density value $d(i, j, k)$ is calculated as:
    $$ d(i, j, k) := \sum_{l=0}^{1} \sum_{m=0}^{1} \sum_{n=0}^{1} v(2i+l, 2j+m, 2k+n) $$
    where $v$ is the binary value of the slicemap at the specified coordinates [S1, unofficial].
3.  **Storage:** The resulting integer sum is stored in $d(i, j, k)$.

**Parameters:**
- **Kernel Size:** A $2 \times 2 \times 2$ kernel is used for speed [S1, unofficial]. Larger kernels are possible via dyadic iteration but are not the primary focus of the base method.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def box_filter_density_estimation(slicemap_v, output_dim_x, output_dim_y, output_dim_z):
    """
    Computes density map from binary slicemap using 2x2x2 box filtering.
    
    Args:
        slicemap_v: 3D list/array of binary values (0 or 1).
                    Expected dimensions: (2*output_dim_x, 2*output_dim_y, 2*output_dim_z).
        output_dim_x, output_dim_y, output_dim_z: Dimensions of the resulting density map.
        
    Returns:
        density_map: 3D list/array of integers.
    """
    # Initialize density map with zeros
    density_map = [[[0 for _ in range(output_dim_z)] for _ in range(output_dim_y)] for _ in range(output_dim_x)]
    
    # Iterate over each voxel in the output density grid
    for i in range(output_dim_x):
        for j in range(output_dim_y):
            for k in range(output_dim_z):
                # Sum the 2x2x2 block from the input slicemap
                count = 0
                for l in range(2):
                    for m in range(2):
                        for n in range(2):
                            # Access the corresponding voxel in the slicemap
                            # Note: Indices must be within bounds of slicemap_v
                            val = slicemap_v[2*i + l][2*j + m][2*k + n]
                            count += val
                # Store the count in the density map
                density_map[i][j][k] = count
                
    return density_map
```

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** The input slicemap dimensions must be even in all axes to allow perfect dyadic downsampling without partial blocks. If dimensions are odd, padding or truncation is required [G].
- **Precision:** Since the input is binary and the output is an integer count, no floating-point precision issues arise during the summation. The maximum value is 8 for a 2x2x2 kernel.

## Subtopics

### [Parallel Bitwise Density Computation](9127-parallel-bitwise-density-computation.md)
Uses parallel bitwise operations on integer textures to compute density, avoiding expensive texture lookups for bit counting. It sets itself apart by using exact integer arithmetic on binary slicemaps rather than smoothed approximations. Choose this when GPU parallelism and bitwise efficiency are prioritized over simple summation.

### [Kernel Density Estimation for Volume Reconstruction](9516-kernel-density-estimation-for-volume-reconstruction.md)
Reconstructs a 3D volume from discrete samples by treating reconstruction as a density estimation problem, allowing samples to be splatted independently. Choose this when smooth, probabilistic density fields are required rather than exact voxel counts.

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of voxels in the output density map. Each output voxel requires a constant number of operations (8 reads and 7 adds for a 2x2x2 kernel).
- **Memory Complexity:** $O(N)$ for the output density map. The input slicemap must be accessible.
- **Performance:** Optimized for speed on GPU architectures due to the simplicity of the box filter and dyadic downsampling, which aligns well with texture memory access patterns [S1, unofficial].
- **Strengths:** Fast, simple, exact integer counts, easy to parallelize.
- **Limitations:** Produces a blocky, non-smooth density field compared to kernel-based methods. Loses high-frequency detail within the 2x2x2 block.
- **When to use:** When computational speed is critical and a binary or low-resolution density representation is sufficient.
- **When not to use:** When smooth, continuous density gradients are required for rendering or simulation.

## Background
The method originates from single-pass GPU solid voxelization techniques, where efficient construction of density maps from binary slicemaps is necessary for applications like ray marching or collision detection [S1, unofficial]. The approach leverages the dyadic nature of downsampling to simplify the filtering process, making it suitable for real-time graphics applications.

## Sources
[S1] Single-pass GPU Solid Voxelization and Applications, https://inria.hal.science/inria-00345291, fetched 2026-10-26 (unofficial)
