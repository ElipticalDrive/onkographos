# Box-filtering Density Estimation


## Overview
Box-filtering density estimation transforms a binary slicemap into a volumetric density representation by filtering and downsampling the grid [S1, unofficial]. Each resulting voxel stores the count of filled voxels from the original high-resolution grid that it represents [S1, unofficial]. It sets itself apart from sibling methods like Kernel Density Estimation by using a simple sum of neighbors for speed, rather than complex kernel weights or probabilistic approximations [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A binary slicemap $v$, where $v(x, y, z) \in \{0, 1\}$ represents the presence of geometry in the original high-resolution grid.
*   **Output:** A downsampled density map $d$, where $d(i, j, k)$ is an integer representing the density.

**Data Structures:**
*   $v$: A 3D array of binary values (uint8 or bit-packed).
*   $d$: A 3D array of integers (uint16 or uint32, depending on max kernel size). Dimensions are half the resolution of $v$ (assuming power-of-two downsampling).

**Parameters:**
*   **Kernel Size:** The method supports any power-of-two kernel size. The primary example uses a $2 \times 2 \times 2$ kernel [S1, unofficial].
*   **Filter Type:** Box filter (uniform weights) [S1, unofficial].

**Steps:**
1.  **Define Grids:** Let the original slicemap $v$ have dimensions $(N_x, N_y, N_z)$. The density map $d$ will have dimensions $(N_x/2, N_y/2, N_z/2)$ [S1, unofficial].
2.  **Iterate:** For each voxel $(i, j, k)$ in the density map $d$:
3.  **Sum Neighbors:** Compute the sum of the $2 \times 2 \times 2$ block of voxels in $v$ corresponding to $(i, j, k)$.
4.  **Store:** Assign the sum to $d(i, j, k)$.

**Formula:**
The density value $d(i, j, k)$ is computed as:

$$d(i, j, k) := \sum_{l=0}^{1} \sum_{m=0}^{1} \sum_{n=0}^{1} v(2i+l, 2j+m, 2k+n)$$

where $v$ is the binary value of the slicemap [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def box_filter_density_estimation(v, nx, ny, nz):
    """
    Computes density map from binary slicemap using 2x2x2 box filtering.
    
    Args:
        v: 3D list/array of shape (nx, ny, nz) containing 0s and 1s.
        nx, ny, nz: Dimensions of the input slicemap.
        
    Returns:
        d: 3D list/array of shape (nx/2, ny/2, nz/2) containing integer densities.
    """
    # Ensure dimensions are even for simple 2x2x2 downsampling
    if nx % 2 != 0 or ny % 2 != 0 or nz % 2 != 0:
        raise ValueError("Input dimensions must be even for 2x2x2 box filtering.")
        
    dx, dy, dz = nx // 2, ny // 2, nz // 2
    d = [[0] * dz for _ in range(dy)]
    
    for i in range(dx):
        for j in range(dy):
            for k in range(dz):
                # Sum the 2x2x2 block in the original slicemap
                density = 0
                for l in range(2):
                    for m in range(2):
                        for n in range(2):
                            density += v[2*i + l][2*j + m][2*k + n]
                d[j][k] = density # Note: Indexing order depends on storage layout (row-major vs col-major)
                
    return d
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** The formula assumes the input grid dimensions are divisible by the kernel size (specifically 2 for the standard case). If dimensions are not even, boundary handling is not specified in the source [S1, unofficial].
*   **Precision:** Since the input is binary and the kernel is small ($2^3=8$), the maximum density value is 8. `uint8` is sufficient for storage, but `uint16` or `uint32` is safer for generalization to larger kernels [S1, unofficial].
*   **Parallelism:** The method is designed for GPU implementation, respecting parallelism and limited memory by processing independent blocks [S1, unofficial].

## Subtopics

### [Parallel Bitwise Density Computation](9127-parallel-bitwise-density-computation.md)
Uses parallel bitwise operations on integer textures to count filled voxels, avoiding texture lookups. Chosen when exact integer arithmetic on binary slicemaps is preferred over smoothing.

### [Kernel Density Estimation for Volume Reconstruction](9516-kernel-density-estimation-for-volume-reconstruction.md)
Reconstructs volume from discrete samples using kernel functions. Chosen when smooth, probabilistic density fields are needed rather than hard counts.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels in the output density map. Each output voxel requires a constant number of operations (8 reads and 7 adds for a $2 \times 2 \times 2$ kernel).
*   **Memory Complexity:** $O(N)$ for the output density map. The input slicemap must be accessible.
*   **Performance:** Optimized for speed on GPU hardware. The box filter is computationally inexpensive compared to other kernels [S1, unofficial].
*   **Strengths:** Simple implementation, fast execution, suitable for parallel processing.
*   **Limitations:** Produces a blocky density representation compared to smoother kernels. Requires power-of-two downsampling for the standard dyadic approach [S1, unofficial].
*   **When to Use:** When speed is the primary concern and a binary-to-density conversion is needed for further processing (e.g., solid voxelization) [S1, unofficial].
*   **When Not to Use:** When smooth, continuous density gradients are required for rendering or simulation.

## Background
The method was introduced in the context of single-pass GPU solid voxelization [S1, unofficial]. It addresses the need to efficiently compute density from a binary slicemap while respecting GPU parallelism and memory constraints [S1, unofficial]. The approach leverages the simplicity of box filtering to achieve speed, contrasting with more complex filtering strategies discussed in broader literature [S1, unofficial].

## Sources
[S1] Single-pass GPU Solid Voxelization and Applications, https://inria.hal.science/inria-00345291, fetched 2026-10-26 (unofficial)
