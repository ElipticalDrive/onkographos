# Voxel Trilinear Interpolation (SIMD)

Also called: SIMD Trilinear Sampling, Trilinear Interpolation with AVX, Trilinear Volume Interpolation, Trilinear Voxel Interpolation

## Overview
Voxel Trilinear Interpolation (SIMD) retrieves scalar or vector voxel data at arbitrary normalized coordinates $(u,v,w)$ by performing trilinear interpolation between the 8 surrounding voxel corners, optimized using Single Instruction, Multiple Data (SIMD) instructions such as AVX [S1, unofficial][S3, unofficial]. It builds on [Grid Cell Collection via Corner Indices](1518-grid-cell-collection-via-corner-indices.md) by providing the arithmetic core for blending those collected values [S1, unofficial]. Unlike standard scalar trilinear interpolation, this method processes multiple data channels or components in parallel using vector registers, significantly increasing throughput for volume rendering and ray marching [S3, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A 3D query point $(x, y, z)$, typically normalized or in grid coordinates; a volume data structure (dense 1D array or brick) containing voxel values; volume dimensions $(D_x, D_y, D_z)$ or brick dimensions.
- **Output:** The interpolated value (scalar or vector) at the query point.

**Data Structures:**
- **Volume Data:** Stored in a 1D array in row-major order. For a brick of size $(B_x, B_y, B_z)$, the data size is $(B_x+1) \times (B_y+1) \times (B_z+1)$ [S3, unofficial]. The linear index for a voxel at $(i, j, k)$ is calculated as:
  $$ \text{idx} = (k \cdot (B_y+1) + j) \cdot (B_x+1) + i $$
  Strides are defined as: $S_x = 1$, $S_y = B_x+1$, $S_z = (B_x+1)(B_y+1)$ [S3, unofficial].
- **SIMD Registers:** 128-bit or 256-bit registers (e.g., AVX `__m256`) used to hold multiple float components simultaneously [S3, unofficial].

**Numbered Steps:**
1. **Coordinate Normalization and Indexing:**
   - If input coordinates are world-space, convert to grid indices. Let the relative position within the brick be $r = (x, y, z) - \text{bmin}$ [S3, unofficial].
   - Normalize relative position to voxel indices: $tmp = r / \text{brick\_size}$ [S3, unofficial].
   - Calculate integer indices and fractional parts:
     $$ i = \lfloor tmp \rfloor, \quad f = tmp - i $$
     where $i = (i_x, i_y, i_z)$ and $f = (f_x, f_y, f_z)$ [S3, unofficial].
   - **Boundary Check:** If any component of $i$ is $< 0$ or $\ge$ dimension, return 0.0 or handle clamping [S3, unofficial]. For clamping, indices are bounded by $[0, \text{dim}-1]$ [S2, unofficial].

2. **Corner Fetching:**
   - Retrieve the 8 corner values from the 1D array using the base index and strides $S_x, S_y, S_z$ [S3, unofficial]:
     $$
     \begin{aligned}
     v_{000} &= \text{data}[idx] \\
     v_{001} &= \text{data}[idx + S_z] \\
     v_{010} &= \text{data}[idx + S_y] \\
     v_{011} &= \text{data}[idx + S_y + S_z] \\
     v_{100} &= \text{data}[idx + S_x] \\
     v_{101} &= \text{data}[idx + S_x + S_z] \\
     v_{110} &= \text{data}[idx + S_x + S_y] \\
     v_{111} &= \text{data}[idx + S_x + S_y + S_z]
     \end{aligned}
     $$
     where $v_{ijk}$ corresponds to the corner at $(i_x+i, i_y+j, i_z+k)$ [S3, unofficial].

3. **SIMD Interpolation:**
   - Broadcast fractional coordinates into SIMD registers. Let $f_x, f_y, f_z$ be the fractional parts [S3, unofficial].
   - Compute weights:
     $$ w_x = f_x, \quad w_y = f_y, \quad w_z = f_z $$
     $$ \bar{w}_x = 1 - w_x, \quad \bar{w}_y = 1 - w_y, \quad \bar{w}_z = 1 - w_z $$
   - **Vectorized Calculation:**
     - Load corner values into SIMD registers. For AVX, load 4 floats at a time.
     - Perform interpolation along X, then Y, then Z using vector multiply-add operations [S3, unofficial].
     - Alternatively, compute the full trilinear product directly if data width matches register width:
       $$ V = \sum_{i,j,k \in \{0,1\}} v_{ijk} \cdot \bar{w}_x^{1-i} w_x^i \cdot \bar{w}_y^{1-j} w_y^j \cdot \bar{w}_z^{1-k} w_z^k $$
     - A specific AVX implementation splits data into low/high 128-bit parts, computes contributions using broadcasted weights, and sums them [S3, unofficial].

4. **Final Output:**
   - Return the interpolated value. If a transfer function is applied, cast the result to the appropriate type (e.g., `uchar`) and sample the function [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_trilinear_simd(data, dims, brick_size, x, y, z):
    """
    data: 1D array of floats
    dims: tuple (Dx, Dy, Dz)
    brick_size: tuple (Bx, By, Bz)
    x, y, z: float coordinates within the volume
    """
    # 1. Boundary Check
    if x < 0 or x >= dims[0] or y < 0 or y >= dims[1] or z < 0 or z >= dims[2]:
        return 0.0

    # 2. Normalize to brick-local coordinates
    # Assuming x,y,z are already in brick-local space or normalized [0,1]
    # If normalized [0,1], scale by brick_size
    # For this example, assume x,y,z are relative to brick origin [0, brick_size]
    
    # 3. Calculate indices and fractions
    ix = int(x)
    iy = int(y)
    iz = int(z)
    
    fx = x - ix
    fy = y - iy
    fz = z - iz
    
    # Clamp indices to valid range [0, dim-1] for corners
    # Note: ix+1 might be out of bounds if x is exactly at the last grid line
    # Standard practice: clamp to max index
    ix0 = max(0, min(ix, dims[0] - 1))
    ix1 = max(0, min(ix + 1, dims[0] - 1))
    iy0 = max(0, min(iy, dims[1] - 1))
    iy1 = max(0, min(iy + 1, dims[1] - 1))
    iz0 = max(0, min(iz, dims[2] - 1))
    iz1 = max(0, min(iz + 1, dims[2] - 1))

    # 4. Calculate strides
    sx = 1
    sy = brick_size[0] + 1
    sz = (brick_size[0] + 1) * (brick_size[1] + 1)
    
    # 5. Fetch 8 corners
    # Base index for the lower corner (ix0, iy0, iz0)
    base_idx = (iz0 * sy + iy0) * sx + ix0
    
    v000 = data[base_idx]
    v001 = data[base_idx + sz]
    v010 = data[base_idx + sy]
    v011 = data[base_idx + sy + sz]
    v100 = data[base_idx + sx]
    v101 = data[base_idx + sx + sz]
    v110 = data[base_idx + sx + sy]
    v111 = data[base_idx + sx + sy + sz]

    # 6. SIMD Interpolation (Simulated with scalar for clarity, 
    # but structure reflects vector ops)
    # Weights
    w_x = fx
    w_y = fy
    w_z = fz
    u_x = 1.0 - w_x
    u_y = 1.0 - w_y
    u_z = 1.0 - w_z

    # Interpolate along X
    v00 = u_x * v000 + w_x * v100
    v01 = u_x * v001 + w_x * v101
    v10 = u_x * v010 + w_x * v110
    v11 = u_x * v011 + w_x * v111

    # Interpolate along Y
    v0 = u_y * v00 + w_y * v10
    v1 = u_y * v01 + w_y * v11

    # Interpolate along Z
    result = u_z * v0 + w_z * v1
    
    return result
```

**Parameters:**
- **Block Dimensions:** In CUDA implementations, block dimensions are often fixed to `BLOCK_SIZE = 8`, resulting in `threads = {8, 8, 8}` (512 threads per block) [S2, unofficial].
- **Grid Dimensions:** Calculated based on the number of points $N_p$ and `MAX_THREADS_AXIS = 128` [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** Indices must be clamped to $[0, \text{dim}-1]$ to prevent out-of-bounds memory access [S2, unofficial][S3, unofficial].
- **Precision:** Using SIMD instructions requires careful handling of data alignment and type casting (e.g., float to uchar for transfer functions) [S3, unofficial].
- **Degenerate Input:** If the query point is outside the volume bounds, the method should return 0 or a default value [S3, unofficial].

## Subtopics

### [Voxel Grid Density Estimation with Trilinear Interpolation](3593-voxel-grid-density-estimation-with-trilinear-interpolation.md)
Estimates accumulated density and color along a ray by sampling a 3D voxel grid using trilinear interpolation, then aggregating values using a transmittance-based opacity formula. Distinguished by producing smooth boundaries without lumpy artifacts.

### [Trilinear Interpolation on Sparse Voxel Octree](5118-trilinear-interpolation-on-sparse-voxel-octree.md)
Samples scalar or vector fields stored in the leaf nodes of a Sparse Voxel Octree (SVO) using trilinear interpolation. Navigates the hierarchical tree to access only occupied voxels, suitable for sparse data.

### [Voxel Interpolation (Nearest, Trilinear, Triquadratic, Tricubic)](6486-voxel-interpolation-nearest-trilinear-triquadratic-tricubic.md)
Provides a unified interface for sampling values from a dense or sparse voxel grid using interpolation orders 0, 1, 2, and 3. Exposes a templated API abstracting the underlying tree structure, whereas this method is hardcoded to trilinear with AVX optimizations.

## Operating characteristics
- **Time Complexity:** $O(1)$ per query point, as it involves a fixed number of arithmetic operations and memory accesses (8 corners) [S3, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary space, dependent on the size of the SIMD registers [S3, unofficial].
- **Measured Performance:** Optimized for GPU (CUDA) and CPU (AVX) architectures. The use of SIMD instructions allows processing multiple data channels or components in parallel, significantly increasing throughput compared to scalar implementations [S3, unofficial].
- **Strengths:** High performance due to vectorization; smooth interpolation reduces aliasing artifacts compared to nearest-neighbor sampling [S1, unofficial].
- **Limitations:** Requires contiguous memory layout for efficient SIMD loading; boundary handling adds conditional checks [S3, unofficial].
- **When to Use:** Ideal for real-time volume rendering and ray marching where performance is critical and smooth interpolation is desired [S5, unofficial].
- **When Not to Use:** For highly sparse data where a Sparse Voxel Octree is more memory-efficient [S1, unofficial].

## Background
Voxel Trilinear Interpolation (SIMD) is an optimization of the standard trilinear interpolation technique, adapted for modern SIMD architectures (AVX, NEON) and GPU shaders (CUDA) [S3, unofficial][S2, unofficial]. It is widely used in volume rendering systems to sample density and color fields efficiently [S5, unofficial]. The method is a core component of many interactive volume rendering applications, enabling smooth visualization of 3D datasets such as medical images and scientific simulations [S5, unofficial].

## Sources
[S1] Scrawk/Marching-Cubes, https://github.com/Scrawk/Marching-Cubes, fetched 2026-10-26 (unofficial)
[S2] tatsy/torchmcubes, https://github.com/tatsy/torchmcubes, fetched 2026-10-26 (unofficial)
[S3] hopexn/VolumeRenderingParallel, https://github.com/hopexn/VolumeRenderingParallel, fetched 2026-10-27 (unofficial)
[S4] cgibson/Mr-Voxel, https://github.com/cgibson/Mr-Voxel, fetched 2026-10-27 (unofficial)
[S5] GitHub - yuehaowang/volume_renderer: CUDA-based interactive volume ..., https://github.com/yuehaowang/volume_renderer, fetched 2026-10-29 (unofficial)
