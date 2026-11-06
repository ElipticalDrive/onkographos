# Occupancy Voxel Creation


## Overview
Occupancy Voxel Creation converts a continuous density matrix into a discrete boolean occupancy mask and identifies fully occupied 2x2x2 blocks via convolution [S1, unofficial]. It differs from [3D Voxel Grid Smoothing](833-3d-voxel-grid-smoothing.md) by using binary thresholding and exact block matching rather than averaging, and from [Binary Voxel RLE Encoding](1071-binary-voxel-rle-encoding.md) by producing a structural mask of solid blocks rather than compressing run-lengths.

## How it works
**Inputs:** A 3D density matrix $D$ of shape $(H, W, D)$.
**Outputs:** A boolean mask $M$ of shape $(H-1, W-1, D-1)$ indicating 2x2x2 blocks where all 8 constituent voxels are occupied.

**Data Structures:**
- $D$: Float array, density values.
- $O$: Binary array, $O_{ijk} = 1$ if $D_{ijk} > 0$, else $0$.
- $K$: 3D kernel of ones, shape $(2, 2, 2)$.
- $M$: Boolean array, result of convolution threshold.

**Steps:**
1. **Binary Thresholding:** Convert density to occupancy.
   $$ O_{ijk} = \begin{cases} 1 & \text{if } D_{ijk} > 0 \\ 0 & \text{otherwise} \end{cases} $$
2. **Kernel Definition:** Define a 2x2x2 kernel $K$ where all elements are $1$.
3. **Convolution:** Perform a 3D convolution of $O$ with $K$ using 'SAME' padding and stride 1.
   $$ C = O * K $$
   The output $C$ has shape $(H, W, D)$. Each element $C_{ijk}$ represents the sum of occupied voxels in the 2x2x2 block anchored at $(i, j, k)$.
4. **Mask Extraction:** Create a boolean mask where the convolution sum equals 8 (all voxels in the block are occupied).
   $$ M_{ijk} = (C_{ijk} == 8) $$
   The mask $M$ has shape $(H-1, W-1, D-1)$ if cropped to valid block anchors, or remains $(H, W, D)$ with boundary blocks evaluated against padded zeros.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import jax.numpy as jnp
from jax.lax import conv_general_dilated

def create_occupancy_voxels(density_matrix: np.ndarray) -> np.ndarray:
    """
    Converts a density matrix to a boolean occupancy mask of fully occupied 2x2x2 blocks.
    
    Args:
        density_matrix: 3D numpy array of shape (H, W, D) with density values.
        
    Returns:
        Boolean mask of shape (H-1, W-1, D-1) indicating 2x2x2 blocks where all 8 voxels are occupied.
    """
    # Step 1: Convert density to binary occupancy
    # Threshold at 0: values > 0 become 1, others 0
    occupancy = (density_matrix > 0).astype(np.float32)
    
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
    
    # Step 5: Extract mask where sum equals 8
    # out shape is (1, H, W, D, 1)
    mask = np.array(out.squeeze() == 8)
    
    # Crop to valid 2x2x2 block anchors (H-1, W-1, D-1)
    # Note: 'SAME' padding includes boundary blocks that extend beyond grid.
    # To get only fully internal blocks, crop by 1 on each spatial dimension.
    return mask[:-1, :-1, :-1]
```

**Parameters:**
- Threshold: $0$ [S1, unofficial].
- Kernel size: $2 \times 2 \times 2$ [S1, unofficial].
- Padding: 'SAME' [S1, unofficial].

**Edge Cases:**
- **Boundary Voxels:** Voxels on the grid boundary have fewer than 8 neighbors in a 2x2x2 block. With 'SAME' padding, these are padded with zeros, so boundary blocks will never sum to 8 and will be marked false. This is consistent with requiring a full 2x2x2 block [S1, unofficial].
- **Precision:** Density values are compared to $0$. Floating point noise near zero may cause instability. A small epsilon threshold may be needed in practice [G].

## Subtopics

### [Mark Invisible Cells](4648-mark-invisible-cells.md)
Marks voxels not visible to any training camera by setting occupancy to -1. Chosen when visibility culling is required before training.

### [Occupancy Grid Filling](6913-occupancy-grid-filling.md)
Fills interior regions of an occupancy grid to ensure topological closure. Chosen when hollow or fragmented masks need to be solidified.

## Operating characteristics
- **Time Complexity:** $O(H \times W \times D)$ for thresholding and convolution. Convolution is efficient on GPUs via JAX [S1, unofficial].
- **Memory Complexity:** $O(H \times W \times D)$ for intermediate arrays (occupancy, convolution output).
- **Strengths:** Simple, differentiable-friendly (if using soft thresholding), identifies solid blocks for efficient raymarching [S1, unofficial].
- **Limitations:** Only detects fully occupied blocks; sparse or thin structures may be missed. Boundary effects require careful handling [S1, unofficial].
- **When to use:** When a coarse, block-based occupancy representation is sufficient for rendering or training acceleration.
- **When not to use:** When fine-grained occupancy details are needed, or when the density field is very sparse.

## Background
Originated in the context of JAX-based volumetric rendering pipelines, specifically in the `Jax-Volumetric-Renderer` repository [S1, unofficial]. It is used to create a binary occupancy grid from continuous density fields for efficient voxel traversal and training.

## Sources
[S1] szat/Jax-Volumetric-Renderer, https://github.com/szat/Jax-Volumetric-Renderer, fetched 2026-10-27 (unofficial)
