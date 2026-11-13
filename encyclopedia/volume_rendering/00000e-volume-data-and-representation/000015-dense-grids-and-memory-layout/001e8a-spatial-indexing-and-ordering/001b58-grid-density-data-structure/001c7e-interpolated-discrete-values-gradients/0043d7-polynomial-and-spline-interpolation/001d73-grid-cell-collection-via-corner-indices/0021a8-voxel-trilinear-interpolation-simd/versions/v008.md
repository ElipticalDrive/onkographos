# Voxel Trilinear Interpolation (SIMD)

Also called: SIMD Trilinear Sampling, Trilinear Interpolation with AVX, Trilinear Volume Interpolation, Trilinear Voxel Interpolation

## Overview
Voxel Trilinear Interpolation (SIMD) retrieves scalar or vector values from a dense 3D volume grid at arbitrary floating-point coordinates by performing trilinear interpolation across the 8 surrounding voxel corners, optimized for Single Instruction, Multiple Data (SIMD) execution units [S1, S3, S5, unofficial]. It solves the problem of extracting continuous fields from discrete voxel representations while maximizing throughput on modern CPUs [S1, S3, unofficial]. It sets itself apart from its sibling [3D Trilinear Interpolation](2613-3d-trilinear-interpolation.md) by explicitly structuring the computation to exploit SIMD registers (e.g., AVX/AVX2) for parallel evaluation of multiple components or channels, and from [VoxelGrid](2159-voxelgrid.md) by providing the specific low-level sampling kernel rather than the grid structure itself.

## How it works
**Inputs:**
- A 3D volume data array. Sources describe two common layouts:
    1. A 1D array of `uchar` or `float` values representing a brick of size $(B_x+1) \times (B_y+1) \times (B_z+1)$ in row-major order [S3, unofficial].
    2. A 4D tensor `vol` with shape $(C, N_z, N_y, N_x)$ accessed as `vol[c][z][y][x]` [S2, unofficial].
- Query coordinates $(u, v, w)$ or $(x, y, z)$. These may be normalized $[0, 1]$ or absolute grid indices [S1, S2, unofficial].
- Volume dimensions $(D_x, D_y, D_z)$ or brick dimensions $(B_x, B_y, B_z)$ [S3, unofficial].

**Outputs:**
- Interpolated scalar or vector value(s) at the query point [S1, S3, unofficial].

**Data Structures:**
- **Voxel Data:** Stored contiguously. For SIMD optimization, data is often loaded into 128-bit or 256-bit registers [S3, unofficial].
- **Corner References:** The 8 corners of the cell enclosing the query point are identified by integer indices $(i, j, k)$ where $i,j,k \in \{0, 1\}$ [S1, S5, unofficial].

**Steps:**

1. **Coordinate Conversion and Bounds Checking:**
   - If inputs are normalized $(u,v,w)$, convert to absolute grid coordinates: $x = u \cdot D_x$, $y = v \cdot D_y$, $z = w \cdot D_z$ [S1, S3, unofficial].
   - Some implementations center coordinates by subtracting 0.5: $cx = x - 0.5, cy = y - 0.5, cz = z - 0.5$ [S4, unofficial].
   - Check if the query point is outside the volume bounds. If any coordinate is $< 0$ or $\ge$ dimension size, return a default value (e.g., 0.0) [S3, S5, unofficial].

2. **Index and Fraction Extraction:**
   - Compute integer grid indices for the lower corner:
     $$ i = \lfloor x \rfloor, \quad j = \lfloor y \rfloor, \quad k = \lfloor z \rfloor $$
   - Compute fractional offsets:
     $$ fx = x - i, \quad fy = y - j, \quad fz = z - k $$
   - Clamp indices to valid ranges $[0, D-1]$ to handle boundary conditions safely [S2, S3, unofficial]. Specifically, clamp to $[0, D-2]$ for the lower corner to ensure the upper corner $(i+1, j+1, k+1)$ is within bounds, or clamp all 8 corners individually to $[0, D-1]$ [S2, unofficial].

3. **Corner Value Retrieval:**
   - Calculate the linear base index for the lower corner $(i, j, k)$ using row-major strides:
     $$ \text{idx} = k \cdot (D_y \cdot D_x) + j \cdot D_x + i $$
     (Note: Strides depend on specific layout; [S3, unofficial] uses $xNext=1, yNext=B_x+1, zNext=(B_x+1)(B_y+1)$ for brick data) [S3, unofficial].
   - Retrieve the 8 corner values $f_{ijk}$ from the data array:
     $$
     \begin{aligned}
     f_{000} &= \text{data}[\text{idx}] \\
     f_{100} &= \text{data}[\text{idx} + 1] \\
     f_{010} &= \text{data}[\text{idx} + D_x] \\
     f_{110} &= \text{data}[\text{idx} + D_x + 1] \\
     f_{001} &= \text{data}[\text{idx} + D_x \cdot D_y] \\
     f_{101} &= \text{data}[\text{idx} + D_x \cdot D_y + 1] \\
     f_{011} &= \text{data}[\text{idx} + D_x \cdot D_y + D_x] \\
     f_{111} &= \text{data}[\text{idx} + D_x \cdot D_y + D_x + 1]
     \end{aligned}
     $$
     [S3, S5, unofficial].

4. **Trilinear Interpolation (Scalar):**
   - Perform interpolation along the X-axis for the two Z-levels ($z=0$ and $z=1$):
     $$
     \begin{aligned}
     v_{00} &= (1-fx)f_{000} + fx \cdot f_{100} \\
     v_{01} &= (1-fx)f_{001} + fx \cdot f_{101} \\
     v_{10} &= (1-fx)f_{010} + fx \cdot f_{110} \\
     v_{11} &= (1-fx)f_{011} + fx \cdot f_{111}
     \end{aligned}
     $$
   - Interpolate along the Y-axis for the two Z-levels:
     $$
     \begin{aligned}
     v_0 &= (1-fy)v_{00} + fy \cdot v_{10} \\
     v_1 &= (1-fy)v_{01} + fy \cdot v_{11}
     \end{aligned}
     $$
   - Interpolate along the Z-axis to get the final value:
     $$ V = (1-fz)v_0 + fz \cdot v_1 $$
     [S2, S4, unofficial].

   - **Expanded Formula:**
     $$
     \begin{aligned}
     V = \;& f_{000}(1-fx)(1-fy)(1-fz) + f_{100}fx(1-fy)(1-fz) + \\
          & f_{010}(1-fx)fy(1-fz) + f_{110}fx \cdot fy(1-fz) + \\
          & f_{001}(1-fx)(1-fy)fz + f_{101}fx(1-fy)fz + \\
          & f_{010}(1-fx)fy \cdot fz + f_{111}fx \cdot fy \cdot fz
     \end{aligned}
     $$
     [S3, unofficial].

5. **SIMD Optimization (AVX/AVX2):**
   - Load voxel data into SIMD registers. For example, load 8 bytes (or 4 floats) into a 128-bit or 256-bit register [S3, unofficial].
   - Broadcast fractional weights. For instance, broadcast $fx$ to all lanes [S3, unofficial].
   - Compute interpolation weights in parallel. Calculate terms like $(1-fy)$ and $(1-fz)$ and combine them with $fx$ using SIMD multiply/add instructions [S3, unofficial].
   - Example AVX2 approach for 8-byte data:
     - Load low and high parts of the voxel data into `v128_l` and `v128_h` [S3, unofficial].
     - Compute high part contribution: `tmp1 = v128_h * fx`, `tmp2 = weights_yz`, `res = tmp1 * tmp2` [S3, unofficial].
     - Compute low part contribution: `fx_inv = 1 - fx`, `tmp1 = v128_l * fx_inv`, `res += tmp1 * tmp2` [S3, unofficial].
     - Sum components within the register to produce the final scalar result [S3, unofficial].

**Parameters:**
- **Block Dimensions:** In CUDA implementations, fixed block sizes like $8 \times 8 \times 8$ (512 threads) are used [S2, unofficial].
- **Brick Size:** For tiled volumes, brick dimensions $(B_x, B_y, B_z)$ determine the local grid size [S3, unofficial].

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** Indices must be clamped to $[0, D-1]$ to prevent out-of-bounds memory access. Failure to clamp results in undefined behavior or crashes [S2, S3, unofficial].
- **Precision:** Using `float` is standard; `double` may be used for higher precision but reduces SIMD throughput [S2, unofficial].
- **Degenerate Input:** Query points exactly on grid boundaries or outside the volume must be handled explicitly (clamping or returning default values) [S3, unofficial].

## Subtopics

### [Voxel Grid Density Estimation with Trilinear Interpolation](3593-voxel-grid-density-estimation-with-trilinear-interpolation.md)
Estimates accumulated density and color along a ray by sampling a 3D voxel grid using trilinear interpolation. Distinguished by providing higher-level ray marching logic and boundary handling.

### [Trilinear Interpolation on Sparse Voxel Octree](5118-trilinear-interpolation-on-sparse-voxel-octree.md)
Samples fields stored in SVO leaf nodes. Distinguished by navigating the octree structure to find neighbors rather than using a dense grid index.

### [Voxel Interpolation (Nearest, Trilinear, Triquadratic, Tricubic)](6486-voxel-interpolation-nearest-trilinear-triquadratic-tricubic.md)
Supports multiple interpolation orders (0–3). Distinguished by offering a choice of polynomial orders and integration with high-level frameworks like PyTorch.

## Operating characteristics
- **Time Complexity:** $O(1)$ per query point, as it involves a fixed number of arithmetic operations and memory accesses (8 samples) [S1, S3, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary space per query, assuming the volume data is already in memory [S3, unofficial].
- **Performance:** SIMD optimization significantly increases throughput by processing multiple data elements (e.g., 4 floats or 8 bytes) in parallel per CPU cycle [S3, unofficial]. Measured performance depends on CPU architecture (AVX2 vs AVX-512) and memory bandwidth [S3, unofficial].
- **Strengths:** High performance on CPUs, smooth interpolation results, simple implementation [S1, S3, unofficial].
- **Limitations:** Requires dense memory layout for optimal SIMD efficiency; boundary handling adds branching or clamping logic; not suitable for sparse data without additional indexing structures [S3, S5, unofficial].
- **When to use:** When sampling dense voxel grids on CPUs with high throughput requirements [S3, unofficial].
- **When not to use:** For sparse volumes (use SVO methods) or when GPU acceleration is preferred (use CUDA kernels) [S2, S3, unofficial].

## Background
Voxel trilinear interpolation is a fundamental technique in volume rendering and computer graphics for sampling continuous fields from discrete voxel data [S1, S5, unofficial]. The SIMD-optimized variant leverages CPU vector instructions (such as Intel AVX/AVX2) to accelerate the interpolation process, which is critical for real-time or high-performance volume rendering applications [S3, unofficial]. It is widely used in medical imaging, scientific visualization, and procedural texture generation [S1, S5, unofficial]. The method builds on the general concept of trilinear interpolation, extending it to exploit parallel processing capabilities of modern processors [S3, unofficial].

## Sources
[S1] Scrawk/Marching-Cubes, https://github.com/Scrawk/Marching-Cubes, fetched 2026-10-26 (unofficial)
[S2] tatsy/torchmcubes, https://github.com/tatsy/torchmcubes, fetched 2026-10-26 (unofficial)
[S3] hopexn/VolumeRenderingParallel, https://github.com/hopexn/VolumeRenderingParallel, fetched 2026-10-27 (unofficial)
[S4] cgibson/Mr-Voxel, https://github.com/cgibson/Mr-Voxel, fetched 2026-10-27 (unofficial)
[S5] GitHub - yuehaowang/volume_renderer: CUDA-based interactive volume ..., https://github.com/yuehaowang/volume_renderer, fetched 2026-10-29 (unofficial)
