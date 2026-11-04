# Occupancy Mipmap Generation
## Overview
Occupancy Mipmap Generation computes a hierarchical occupancy mipmap for a 3D grid using JAX convolutions [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: `mat`, a 3D array of shape `(xdim, ydim, zdim)` containing density or occupancy values [S1, unofficial].
- Output: `mipmap`, a list of JAX arrays representing the occupancy pyramid levels, followed by a scalar tensor containing the total number of occupied points [S1, unofficial].

**Data Structures:**
- `occupancy_mat`: A 3D array of shape `(xdim, ydim, zdim)` with `float32` values, initialized to zeros and set to `1.0` where `mat > 0` [S1, unofficial].
- `data`: A 4D JAX array of shape `(1, xdim, ydim, zdim, 1)` created by adding batch and channel dimensions to `occupancy_mat` [S1, unofficial].
- `kernel`: A 5D JAX array of shape `(2^i, 2^i, 2^i, 1, 1)` for level `i` [S1, unofficial].

**Steps:**
1. Calculate the number of mipmap levels: $L = \lfloor \log_2(\text{xdim}) \rfloor$ [S1, unofficial].
2. Create the base occupancy mask: Initialize `occupancy_mat` with zeros. Set `occupancy_mat[x, y, z] = 1.0` if `mat[x, y, z] > 0`, otherwise `0.0` [S1, unofficial].
3. Convert to JAX array and reshape: Cast `occupancy_mat` to `float32` and add batch and channel dimensions to form `data` of shape `(1, xdim, ydim, zdim, 1)` [S1, unofficial].
4. Initialize `mipmap` as an empty list.
5. For each level $i$ from $0$ to $L-1$:
   a. Create a kernel of ones with shape $(2^i, 2^i, 2^i, 1, 1)$ [S1, unofficial].
   b. Define convolution dimension numbers for input `(NHWDC)` and kernel `(HWDIO)` to output `(NHWDC)` [S1, unofficial].
   c. Apply `lax.conv_general_dilated` with stride $(2^i, 2^i, 2^i)$ and padding `'SAME'` [S1, unofficial].
   d. Remove batch and channel dimensions from the result [S1, unofficial].
   e. Append the resulting 3D array to `mipmap` [S1, unofficial].
6. Compute the total number of occupied points: $N = \sum \text{occupancy\_mat}$ [S1, unofficial].
7. Append $N$ as a scalar tensor to `mipmap` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import jax.numpy as jnp
from jax.lax import conv_general_dilated, conv_dimension_numbers

def generate_occupancy_mipmap(mat):
    xdim, ydim, zdim = mat.shape
    levels = int(jnp.log2(xdim))
    
    occupancy_mat = jnp.zeros((xdim, ydim, zdim), dtype=jnp.float32)
    occupancy_mat = jnp.where(mat > 0, 1.0, occupancy_mat)
    
    data = occupancy_mat[None, :, :, :, None]
    mipmap = []
    
    for i in range(levels):
        kernel_size = 2**i
        kernel = jnp.ones((kernel_size, kernel_size, kernel_size, 1, 1))
        dn = conv_dimension_numbers(data.shape, kernel.shape, ('NHWDC', 'HWDIO', 'NHWDC'))
        
        conv_out = conv_general_dilated(
            data, kernel, window_strides=(kernel_size, kernel_size, kernel_size),
            padding='SAME', dimension_numbers=dn
        )
        
        level_grid = conv_out[0, :, :, :, 0]
        mipmap.append(level_grid)
        
    total_occupied = jnp.sum(occupancy_mat)
    mipmap.append(total_occupied)
    return mipmap
```

**Parameters:**
- `mat`: Input 3D grid array. Dimensions `xdim`, `ydim`, `zdim` are used to determine the number of levels [S1, unofficial].

**Edge Cases and Pitfalls:**
- The number of levels is computed as $\lfloor \log_2(\text{xdim}) \rfloor$, which assumes `xdim` is a power of 2 for exact downsampling [S1, unofficial].
- The kernel size grows exponentially as $2^i$, which increases memory requirements for the convolution at higher levels [S1, unofficial].

## Subtopics

### [Occupancy Grid Update](2916-occupancy-grid-update.md)
Periodically refreshes a single-resolution 3D occupancy grid by querying a NeRF model for density values at voxel centers, marking voxels as occupied if density exceeds a threshold. It differs from this method by using direct thresholding on a single resolution rather than hierarchical convolutional smoothing.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:** Not given in the sources.
- **Limitations:** Not given in the sources.
- **When to Use:** Not given in the sources.
- **When Not to Use:** Not given in the sources.

## Background
This method is implemented in the Jax-Volumetric-Renderer project by szat, which uses JAX for volume rendering [S1, unofficial].

## Sources
[S1] szat/Jax-Volumetric-Renderer, https://github.com/szat/Jax-Volumetric-Renderer, fetched 2026-10-27 (unofficial)
