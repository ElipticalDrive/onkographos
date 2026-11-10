# Occupancy Voxel Creation
Also called: block occupancy detection, 2x2x2 block convolution

## Overview
Occupancy Voxel Creation converts a continuous density matrix into a sparse boolean occupancy mask by identifying fully occupied 2x2x2 voxel blocks. It solves the problem of high memory usage in dense voxel grids by collapsing contiguous solid regions into single representative blocks, thereby reducing the number of primitives processed during rendering. Unlike [3D Voxel Grid Smoothing](833-3d-voxel-grid-smoothing.md) which modifies density values, or [Binary Voxel RLE Encoding](1071-binary-voxel-rle-encoding.md) which compresses the final grid, this method performs a structural reduction based on local block occupancy. It builds on [Voxel Grid Optimization](4131-voxel-grid-optimization.md) by implementing spatial culling through block detection.

## How it works
**Inputs and Outputs:**
- Input: `density_mat`, a 3D array of shape $(H, W, D)$ containing continuous density values.
- Output: `block_mask`, a 3D boolean array of shape $(H-1, W-1, D-1)$ indicating the presence of fully occupied 2x2x2 blocks.

**Data Structures:**
- `density_mat`: Floating-point array. Values $> 0$ are considered occupied.
- `block_mask`: Boolean array. `True` at index $(i, j, k)$ if the 2x2x2 block starting at $(i, j, k)$ is fully occupied.

**Steps:**
1.  **Binary Conversion**: Convert the density matrix to a binary occupancy grid. Any voxel with a density value greater than 0 is set to 1; otherwise, it is 0.
    $$ \text{occupancy}[i,j,k] = \begin{cases} 1 & \text{if } \text{density}[i,j,k] > 0 \\ 0 & \text{otherwise} \end{cases} $$
2.  **Kernel Definition**: Define a 2x2x2 convolution kernel consisting entirely of ones. The kernel shape is $(2, 2, 2)$.
3.  **Convolution**: Perform a 3D convolution of the binary occupancy grid with the 2x2x2 kernel. Use 'SAME' padding to maintain spatial dimensions. The stride is $(1, 1, 1)$.
    $$ \text{conv\_out}[i,j,k] = \sum_{di=0}^{1} \sum_{dj=0}^{1} \sum_{dk=0}^{1} \text{occupancy}[i+di, j+dj, k+dk] $$
4.  **Block Identification**: Create a mask where the convolution output equals 8. Since the kernel sums 8 voxels, a value of 8 indicates all 8 voxels in the 2x2x2 block are occupied (value 1).
    $$ \text{block\_mask}[i,j,k] = (\text{conv\_out}[i,j,k] == 8) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import jax.numpy as jnp
from jax.lax import conv_general_dilated

def create_occupancy_voxels(density_mat):
    """
    Converts a density matrix into a boolean occupancy mask of 2x2x2 blocks.
    
    Args:
        density_mat: np.ndarray of shape (H, W, D), continuous density values.
        
    Returns:
        block_mask: np.ndarray of shape (H-1, W-1, D-1), boolean mask.
    """
    # Step 1: Convert density to binary occupancy
    # Values > 0 are occupied (1), others are empty (0)
    occupancy = (density_mat > 0).astype(np.float32)
    
    # Step 2: Prepare data for JAX convolution
    # Add batch and channel dimensions: (1, H, W, D, 1)
    data = occupancy[None, :, :, :, None]
    
    # Step 3: Define 2x2x2 kernel of ones
    # Shape: (2, 2, 2, 1, 1) for JAX conv_general_dilated
    kernel = jnp.ones([2, 2, 2])[:, :, :, jnp.newaxis, jnp.newaxis]
    
    # Step 4: Perform convolution
    # Padding: 'SAME'
    # Stride: (1, 1, 1)
    # Dilation: (1, 1, 1)
    out = conv_general_dilated(
        data, 
        kernel, 
        window_strides=(1, 1, 1), 
        padding='SAME', 
        lhs_dilation=(1, 1, 1), 
        rhs_dilation=(1, 1, 1), 
        dimension_numbers=('NHWC', 'OIHW', 'NHWC') # Standard JAX conv dims
    )
    
    # Step 5: Extract mask where sum equals 8 (all 8 voxels occupied)
    # Remove batch and channel dimensions
    mask = np.array(out.squeeze() == 8)
    
    return mask
```

**Edge Cases and Pitfalls:**
- **Boundary Conditions**: The output mask has dimensions $(H-1, W-1, D-1)$. Voxels on the outermost boundary of the original grid cannot form a complete 2x2x2 block starting at their index, so they are excluded from the mask. This is handled naturally by the convolution output size with 'SAME' padding if the kernel is applied correctly, but the logical mask only covers valid block origins.
- **Precision**: The comparison `== 8` is safe because the input is binary (0 or 1) and the kernel is integer ones, resulting in exact integer sums. No floating-point tolerance is needed.
- **Empty Grids**: If the density matrix is all zeros, the output mask will be all `False`.

## Subtopics

### [Mark Invisible Cells](4648-mark-invisible-cells.md)
Marks voxels invisible to training cameras by setting their occupancy to -1. Chosen when visibility culling is required before training. Differs by focusing on camera visibility rather than spatial block density.

### [Occupancy Grid Filling](6913-occupancy-grid-filling.md)
Fills interior voids in the occupancy grid to ensure topological closure. Chosen when hollow or fragmented masks need to be solidified. Differs by modifying topology rather than detecting blocks.

## Operating characteristics
- **Time Complexity**: $O(H \times W \times D)$ for the binary conversion and $O(H \times W \times D)$ for the convolution (assuming fixed kernel size). The convolution is the dominant operation.
- **Memory Complexity**: $O(H \times W \times D)$ for the input and intermediate convolution output. The output mask is $O((H-1)(W-1)(D-1))$.
- **Performance**: Highly efficient on GPU due to the use of optimized convolution primitives (e.g., JAX `conv_general_dilated`). The method reduces the number of active voxels significantly for dense scenes, leading to faster rendering times in subsequent steps.
- **Strengths**: Simple to implement, leverages standard convolution operations, effective for reducing memory footprint in dense volumes.
- **Limitations**: Only detects fully occupied blocks; partially occupied 2x2x2 regions are ignored, which may lead to loss of detail in thin structures. The output resolution is reduced by 1 in each dimension.

## Background
The method originates from the Jax-Volumetric-Renderer project, which implements efficient volumetric rendering using JAX. It was designed to optimize the voxel grid representation by collapsing dense regions into blocks, thereby reducing the computational load during ray marching. The approach is part of a broader trend in neural rendering to use sparse and hierarchical data structures for efficient volume representation.

## Sources
[S1] Jax-Volumetric-Renderer, https://github.com/szat/Jax-Volumetric-Renderer, fetched 2026-10-27 (unofficial)
