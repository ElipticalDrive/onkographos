# Occupancy Voxel Creation
## Overview
Occupancy Voxel Creation converts a continuous density matrix into a discrete boolean occupancy mask and identifies fully occupied 2x2x2 blocks using convolution [S1, unofficial]. It serves as a preprocessing step for sparse volume rendering by reducing the active voxel domain to dense clusters, differing from siblings like [Binary Voxel RLE Encoding](1071-binary-voxel-rle-encoding.md) by focusing on block-level density detection rather than compression [S1, unofficial].
## How it works
**Inputs:** A 3D density matrix $D$ of shape $(H, W, D)$ containing non-negative float values.
**Outputs:** A boolean mask $M$ of shape $(H-1, W-1, D-1)$ indicating the top-left corner of each fully occupied 2x2x2 block.

**Data Structures:**
- `density_mat`: 3D array of floats.
- `occupancy_mat`: 3D array of integers (0 or 1).
- `kernel`: 5D array for JAX convolution, shape $(2, 2, 2, 1, 1)$.
- `out`: 5D array result of convolution.

**Steps:**
1.  **Binary Conversion:** Convert the density matrix to a binary occupancy mask where any positive density is considered occupied:
    $$
    O[i, j, k] = \begin{cases} 1 & \text{if } D[i, j, k] > 0 \\ 0 & \text{otherwise} \end{cases}
    $$
2.  **Reshape for Convolution:** Reshape the binary occupancy matrix $O$ to add batch and channel dimensions required by the convolution library. If $O$ has shape $(H, W, D)$, the input data $X$ becomes shape $(1, H, W, D, 1)$ [S1, unofficial].
3.  **Kernel Definition:** Define a 2x2x2 kernel of ones. The kernel $K$ has shape $(2, 2, 2, 1, 1)$ where all elements are 1.0 [S1, unofficial].
4.  **Convolution:** Perform a 3D convolution of $X$ with $K$ using 'SAME' padding and stride 1. The output $Y$ has shape $(1, H, W, D, 1)$ [S1, unofficial].
    $$
    Y[b, i, j, k, c] = \sum_{di=0}^{1} \sum_{dj=0}^{1} \sum_{dk=0}^{1} X[b, i+di, j+dj, k+dk, c] \cdot K[di, dj, dk, 0, 0]
    $$
5.  **Mask Extraction:** The convolution sum represents the number of occupied voxels in the 2x2x2 block starting at $(i, j, k)$. A value of 8 indicates all 8 voxels are occupied. Create the final mask $M$ where:
    $$
    M[i, j, k] = (Y[0, i, j, k, 0] == 8)
    $$
    The resulting mask $M$ has shape $(H-1, W-1, D-1)$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import jax.numpy as jnp
from jax.lax import conv_general_dilated

def create_occupancy_voxels(density_mat):
    # Step 1: Convert density to binary occupancy
    # Values > 0 are occupied (1), others are empty (0)
    occupancy_mat = (density_mat > 0).astype(np.float32)
    
    # Step 2: Reshape for JAX convolution
    # Add batch dimension (1) and channel dimension (1)
    # Shape becomes (1, H, W, D, 1)
    data = occupancy_mat[None, :, :, :, None]
    
    # Step 3: Define 2x2x2 kernel of ones
    # Shape (2, 2, 2, 1, 1)
    kernel = jnp.ones([2, 2, 2])[:, :, :, jnp.newaxis, jnp.newaxis]
    
    # Step 4: Perform convolution
    # 'SAME' padding, stride 1, no dilation
    # Output shape: (1, H, W, D, 1)
    out = conv_general_dilated(
        data, 
        kernel, 
        window_strides=(1, 1, 1), 
        padding='SAME', 
        lhs_dilation=(1, 1, 1), 
        rhs_dilation=(1, 1, 1), 
        dimension_numbers=('NHWC', 'OIHW', 'NHWC')
    )
    
    # Step 5: Extract mask where sum equals 8
    # out[0, :, :, :, 0] has shape (H, W, D)
    # Mask has shape (H-1, W-1, D-1) effectively, though array size is HxWxD
    # Note: The last row/col/voxel of the output corresponds to blocks 
    # starting at H-1, W-1, D-1 which are valid. 
    # However, standard convolution with SAME padding on size N with kernel 2 
    # outputs size N. The mask M[i,j,k] represents block at i,j,k.
    # Valid blocks start at 0 to H-2, W-2, D-2.
    # The source code implies direct comparison.
    mask = np.array(out[0, :, :, :, 0] == 8)
    
    # Trim to valid block coordinates if necessary, 
    # though the source returns the full array comparison result.
    # Based on source: mask = np.array(out == 8)
    # We return the boolean mask.
    return mask
```

**Parameters:**
- Density threshold: $> 0$ [S1, unofficial].
- Block size: 2x2x2 [S1, unofficial].

**Edge Cases:**
- **Boundary Conditions:** The convolution output size matches the input size with 'SAME' padding. The mask at indices $(H-1, W-1, D-1)$ checks a block extending beyond the grid if not handled, but the source code returns the full array [S1, unofficial]. Implementers should note that valid 2x2x2 blocks only exist for $i \in [0, H-2], j \in [0, W-2], k \in [0, D-2]$.
- **Precision:** Uses float32 for convolution. Integer comparison `== 8` is robust against floating point noise if the input is strictly binary [S1, unofficial].
- **Empty Volumes:** If the density matrix is all zeros, the output mask is all False [S1, unofficial].

## Subtopics

### [Mark Invisible Cells](4648-mark-invisible-cells.md)
Identifies voxels not covered by any training camera and marks them with density -1. It differs by performing visibility culling rather than density-based block detection.

### [Occupancy Grid Filling](6913-occupancy-grid-filling.md)
Fills interior regions of an occupancy grid by setting background labels to occupied. It differs by focusing on topological closure rather than block detection.

## Operating characteristics
- **Time Complexity:** $O(H \times W \times D)$ for binary conversion and convolution. The convolution is the dominant operation.
- **Memory Complexity:** $O(H \times W \times D)$ for storing the input, output, and kernel.
- **Strengths:** Efficiently identifies dense clusters of voxels, reducing the number of rays that need to be traced in subsequent rendering steps.
- **Limitations:** Only detects fully occupied 2x2x2 blocks, potentially missing sparse but significant geometry.
- **Hardware:** Optimized for GPU execution via JAX convolution primitives [S1, unofficial].

## Background
This method is derived from the Jax-Volumetric-Renderer project, which implements efficient volume rendering using JAX [S1, unofficial]. It is used as a preprocessing step to optimize ray marching by skipping empty or partially empty regions.

## Sources
[S1] szat/Jax-Volumetric-Renderer, https://github.com/szat/Jax-Volumetric-Renderer, fetched 2026-10-27 (unofficial)
