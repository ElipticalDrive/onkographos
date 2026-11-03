# Trilinear Interpolation (Compressed/Linked)

Also called: JAX Trilinear Interpolation (Vmapped)

## Overview
This method performs trilinear interpolation on a compressed sparse grid using a links array to map voxel indices to compressed value indices, handling boundary checks via masking or a zero-value hack [S1]. It builds on [3D Trilinear Interpolation](2613-3d-trilinear-interpolation.md) by adding a link lookup step that translates grid coordinates to compressed indices while retaining the regular 8-corner neighborhood structure.

## How it works
**Inputs:**
- `xyz`: Array of sample coordinates, shape `(N, 3)`, in the same coordinate system as the grid [S1].
- `links`: 3D integer array, shape `(Dx, Dy, Dz)`, mapping each voxel corner to an index in the compressed value array [S1]. Invalid links are represented by negative integers [S1].
- `values_compressed`: Array of values, shape `(M, C)`, where `M` is the number of valid compressed entries and `C` is the channel dimension [S1].

**Outputs:**
- `result`: Interpolated values at the sample points, shape `(N, C)` [S1].

**Data Structures:**
- `links`: A 3D integer array where `links[x, y, z]` returns the index `k` such that `values_compressed[k]` holds the value for the voxel corner at `(x, y, z)` [S1]. If the corner is invalid, `links[x, y, z] < 0` [S1].
- `values_compressed`: A 2D array where each row corresponds to a valid voxel corner's data [S1]. Index 0 is often reserved as a zero-value placeholder for boundary handling in the zero-value hack variant [S1].

**Steps:**

1. **Coordinate Normalization and Decomposition:**
   Compute the floor and fractional parts of the input coordinates.
   $$ x_0, y_0, z_0 = \lfloor xyz \rfloor $$
   $$ xd, yd, zd = xyz - (x_0, y_0, z_0) $$
   Where $x_0, y_0, z_0$ are integer indices for the voxel containing the sample point, and $xd, yd, zd \in [0, 1)$ are the relative positions within that voxel [S1].

2. **Corner Link Lookup:**
   Retrieve the compressed indices for all 8 corners of the voxel defined by $(x_0, y_0, z_0)$.
   $$ l_{ijk} = \text{links}[x_0+i, y_0+j, z_0+k] \quad \text{for } i,j,k \in \{0,1\} $$
   This yields 8 indices: $l_{000}, l_{100}, l_{010}, l_{110}, l_{001}, l_{101}, l_{011}, l_{111}$ [S1].

3. **Validity Masking:**
   Create boolean masks for each link indicating validity (index $\ge 0$) [S1].
   $$ m_{ijk} = (l_{ijk} \ge 0) $$
   For strict mode, compute the intersection of all 8 masks to identify sample points where all 8 corners are valid [S1].
   $$ m_{valid} = m_{000} \land m_{100} \land m_{010} \land m_{110} \land m_{001} \land m_{101} \land m_{011} \land m_{111} $$
   For the zero-value hack variant, compute the union of all 8 masks to identify points touching the boundary [S1].

4. **Boundary Handling (Zero-Value Hack):**
   If using the zero-value hack, replace invalid links (negative indices) within the boundary mask with index 0 before gathering [S1].
   $$ l_{ijk}[m_{boundary} \land \lnot m_{ijk}] = 0 $$

5. **Value Gathering:**
   Gather the values from `values_compressed` using the corresponding links [S1].
   $$ v_{ijk} = \text{values\_compressed}[l_{ijk}] $$

6. **Weight Computation and Interpolation:**
   Compute the trilinear weights based on the fractional coordinates [S1].
   $$ \text{tmpX} = 1 - xd, \quad \text{tmpY} = 1 - yd, \quad \text{tmpZ} = 1 - zd $$
   $$ a_{000} = \text{tmpX} \cdot \text{tmpY}, \quad a_{100} = xd \cdot \text{tmpY}, \quad a_{010} = \text{tmpX} \cdot yd, \quad a_{110} = xd \cdot yd $$
   $$ \text{weights} = [a_{000}, a_{010}, a_{100}, a_{110}] $$
   Gather coefficients and compute the result [S1].
   $$ \text{coeff} = [v_{000}, v_{001}, v_{010}, v_{011}, v_{100}, v_{101}, v_{110}, v_{111}] $$
   $$ \text{result} = \sum \text{weights} \cdot \text{coeff}[[0, 2, 4, 6], :] \cdot \text{tmpZ} + \sum \text{weights} \cdot \text{coeff}[[1, 3, 5, 7], :] \cdot zd $$
   For points where $m_{valid}$ is false in strict mode, the result is zero [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def trilinear_compressed_linked(xyz, links, values_compressed, use_zero_hack=False):
    # 1. Normalize coordinates and compute floor/frac parts
    xyz_floor = np.floor(xyz)
    xd = xyz[:, 0] - xyz_floor[:, 0]
    yd = xyz[:, 1] - xyz_floor[:, 1]
    zd = xyz[:, 2] - xyz_floor[:, 2]
    x0 = xyz_floor[:, 0].astype(int)
    y0 = xyz_floor[:, 1].astype(int)
    z0 = xyz_floor[:, 2].astype(int)

    # 2. Lookup links for all 8 corners
    l000 = links[x0, y0, z0]
    l100 = links[x0 + 1, y0, z0]
    l010 = links[x0, y0 + 1, z0]
    l110 = links[x0 + 1, y0 + 1, z0]
    l001 = links[x0, y0, z0 + 1]
    l101 = links[x0 + 1, y0, z0 + 1]
    l011 = links[x0, y0 + 1, z0 + 1]
    l111 = links[x0 + 1, y0 + 1, z0 + 1]

    # 3. Create masks for valid links (>= 0)
    mask_l000 = l000 >= 0
    mask_l100 = l100 >= 0
    mask_l010 = l010 >= 0
    mask_l110 = l110 >= 0
    mask_l001 = l001 >= 0
    mask_l101 = l101 >= 0
    mask_l011 = l011 >= 0
    mask_l111 = l111 >= 0

    if use_zero_hack:
        # Union mask for boundary points
        mask_boundary = mask_l000 | mask_l100 | mask_l010 | mask_l110 | mask_l001 | mask_l101 | mask_l011 | mask_l111
        # Replace invalid links with 0 within boundary mask
        l000[mask_boundary & ~mask_l000] = 0
        l100[mask_boundary & ~mask_l100] = 0
        l010[mask_boundary & ~mask_l010] = 0
        l110[mask_boundary & ~mask_l110] = 0
        l001[mask_boundary & ~mask_l001] = 0
        l101[mask_boundary & ~mask_l101] = 0
        l011[mask_boundary & ~mask_l011] = 0
        l111[mask_boundary & ~mask_l111] = 0
        mask_valid = mask_boundary
    else:
        # Intersection mask for strict mode
        mask_valid = mask_l000 & mask_l100 & mask_l010 & mask_l110 & mask_l001 & mask_l101 & mask_l011 & mask_l111

    # 4. Gather values
    v000 = values_compressed[l000]
    v100 = values_compressed[l100]
    v010 = values_compressed[l010]
    v110 = values_compressed[l110]
    v001 = values_compressed[l001]
    v101 = values_compressed[l101]
    v011 = values_compressed[l011]
    v111 = values_compressed[l111]

    # 5. Compute weights
    tmpX = 1.0 - xd
    tmpY = 1.0 - yd
    tmpZ = 1.0 - zd
    a000 = tmpX * tmpY
    a100 = xd * tmpY
    a010 = tmpX * yd
    a110 = xd * yd
    weights = np.stack([a000, a010, a100, a110], axis=0)  # Shape: (4, N)

    # 6. Gather coefficients and compute result
    coeff = np.stack([v000, v001, v010, v011, v100, v101, v110, v111], axis=0)  # Shape: (8, C)
    res = np.sum(weights * coeff[[0, 2, 4, 6], :], axis=0) * tmpZ[:, None] + np.sum(weights * coeff[[1, 3, 5, 7], :], axis=0) * zd[:, None]

    # Initialize output and assign valid results
    result = np.zeros_like(res)
    result[mask_valid] = res[mask_valid]
    return result
```

**Subtopics:**
- **Trilinear Interpolation (Compressed/Linked Zero):** A variant where invalid links are replaced with index 0 (assumed to hold a zero value) within a boundary union mask rather than masked out [S1].
- **JAX Trilinear Interpolation (Vmapped):** An implementation strategy using `jax.vmap` for batched processing and `jax.jit` for compilation to accelerate the linked interpolation [S1].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of sample points, with a higher constant factor than dense trilinear due to link lookup and masking operations [S1].
- **Memory Complexity:** $O(D_x D_y D_z)$ for the `links` array, plus $O(M)$ for `values_compressed` [S1].
- **Measured Performance:** Vectorized masking and gathering operations run in parallel; performance is limited by memory bandwidth when accessing `values_compressed` if links are highly scattered [S1].
- **Strengths:** Efficient for sparse data where the bounding box is not excessively large; maintains the smoothness of trilinear interpolation [S1].
- **Limitations:** The `links` array consumes memory proportional to the bounding box volume, not the number of occupied voxels; boundary handling requires careful masking or zero-padding [S1].
- **When to use:** When the volume is sparse but the bounding box is manageable, and smooth interpolation is required [S1].
- **When not to use:** When the bounding box is very large relative to the number of occupied voxels [G].

## Background
This method originates from implementations of volumetric rendering in JAX, specifically for optimizing sparse grid access in neural rendering pipelines [S1]. It leverages the compressed data structure to reduce memory footprint for values while using a link array to maintain spatial locality for interpolation [S1].

## Sources
[S1] szat/Jax-Volumetric-Renderer, https://github.com/szat/Jax-Volumetric-Renderer, fetched 2026-10-27 [S1, unofficial]
