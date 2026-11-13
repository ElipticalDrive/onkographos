# Box-filtering Density Estimation

## Overview
Box-filtering density estimation transforms a binary slicemap into a volumetric density representation by filtering and downsampling the grid [S1, unofficial]. Each resulting voxel stores the count of filled voxels from the original high-resolution grid that it represents [S1, unofficial]. It sets itself apart from sibling methods like Kernel Density Estimation by using a simple sum of neighbors for speed, rather than complex kernel weights or probabilistic approximations [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A binary slicemap $v$, where $v(x, y, z) \in \{0, 1\}$ represents the presence of geometry in the original high-resolution grid [S1, unofficial].
*   **Output:** A downsampled density map $d$, where $d(i, j, k)$ is an integer representing the density count [S1, unofficial].

**Data Structures:**
*   $v$: A 3D array of binary values (uint8 or bit-packed) with dimensions $(N_x, N_y, N_z)$ [S1, unofficial].
*   $d$: A 3D array of integers (uint16 or uint32) with dimensions $(N_x/2, N_y/2, N_z/2)$ for the standard $2\times2\times2$ kernel [S1, unofficial].

**Numbered Steps:**
1.  **Define Grids:** Allocate the output density map $d$ with dimensions half the resolution of $v$ along each axis [S1, unofficial].
2.  **Iterate:** For each voxel $(i, j, k)$ in the output density map $d$, identify the corresponding $2\times2\times2$ block in $v$ [S1, unofficial].
3.  **Sum Neighbors:** Compute the sum of the 8 adjacent binary values in $v$ corresponding to the block [S1, unofficial].
4.  **Store:** Assign the computed sum to $d(i, j, k)$ [S1, unofficial].

**Formula:**
The density value $d(i, j, k)$ is computed as:
$$d(i, j, k) := \sum_{l=0}^{1} \sum_{m=0}^{1} \sum_{n=0}^{1} v(2i+l, 2j+m, 2k+n)$$
where $v$ is the binary value (0 or 1) of the slicemap at the specified coordinates [S1, unofficial].

**Parameters:**
*   **Kernel Size:** The standard implementation uses a $2\times2\times2$ kernel summing 8 neighbors [S1, unofficial]. Any power-of-two kernel size can be obtained dyadically [S1, unofficial].
*   **Filter Type:** Box filter (uniform weights) selected for speed [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** Boundary handling for input grid dimensions not divisible by the kernel size is not specified in the sources.
*   **Precision:** The maximum density value for a $2\times2\times2$ kernel is 8, fitting in `uint8`, though larger kernels require wider integer types [G].
*   **Parallelism:** The method is designed for GPU implementation, respecting parallelism and limited memory [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def box_filter_density_estimation(v, nx, ny, nz):
    """
    Computes density map from binary slicemap using 2x2x2 box filtering.
    Note: The source does not provide pseudocode; this implements the stated formula.
    
    Args:
        v: 3D list/array of shape (nx, ny, nz) containing 0s and 1s.
        nx, ny, nz: Dimensions of the input slicemap.
        
    Returns:
        d: 3D list/array of shape (nx/2, ny/2, nz/2) containing integer densities.
    """
    dx, dy, dz = nx // 2, ny // 2, nz // 2
    # Initialize 3D output array
    d = [[[0] * dz for _ in range(dy)] for _ in range(dx)]
    
    for i in range(dx):
        for j in range(dy):
            for k in range(dz):
                density = 0
                for l in range(2):
                    for m in range(2):
                        for n in range(2):
                            density += v[2*i + l][2*j + m][2*k + n]
                d[i][j][k] = density
    return d
```

## Subtopics

### [Parallel Bitwise Density Computation](9127-parallel-bitwise-density-computation.md)
Computes voxel density using parallel bitwise operations on integer textures, avoiding expensive texture lookups for bit counting. Chosen when exact integer arithmetic on binary slicemaps is preferred over smoothed field approximations.

### [Kernel Density Estimation for Volume Reconstruction](9516-kernel-density-estimation-for-volume-reconstruction.md)
Reconstructs a 3D volume from discrete samples by treating reconstruction as a density estimation problem. Chosen when smooth, probabilistic density fields are needed rather than hard integer counts.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Optimized for speed on GPU hardware; the box filter is computationally inexpensive compared to other kernels [S1, unofficial].
*   **Strengths:** Simple implementation, fast execution, suitable for parallel processing [S1, unofficial].
*   **Limitations:** Produces a blocky density representation compared to smoother kernels; requires power-of-two downsampling for the standard dyadic approach [S1, unofficial].
*   **When to Use:** When speed is the primary concern and a binary-to-density conversion is needed for further processing (e.g., solid voxelization) [S1, unofficial].
*   **When Not to Use:** Not given in the sources.

## Background
The method was introduced in the context of single-pass GPU solid voxelization [S1, unofficial]. It addresses the need to efficiently compute density from a binary slicemap while respecting GPU parallelism and memory constraints [S1, unofficial]. The approach leverages the simplicity of box filtering to achieve speed, contrasting with more complex filtering strategies discussed in broader literature [S1, unofficial].

## Sources
[S1] Single-pass GPU Solid Voxelization and Applications, https://inria.hal.science/inria-00345291, fetched 2026-10-26 (unofficial)
