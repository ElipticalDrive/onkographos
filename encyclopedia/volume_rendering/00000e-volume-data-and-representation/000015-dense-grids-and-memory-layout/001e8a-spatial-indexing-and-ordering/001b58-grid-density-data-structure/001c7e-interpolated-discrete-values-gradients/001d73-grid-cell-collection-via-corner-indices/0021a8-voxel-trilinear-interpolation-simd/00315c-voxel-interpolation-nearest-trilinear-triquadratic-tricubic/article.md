# Voxel Interpolation (Nearest, Trilinear, Triquadratic, Tricubic)


## Overview
Voxel interpolation samples dense volume grids at arbitrary query points using nearest-neighbor, trilinear, triquadratic, or tricubic interpolation, often handling automatic bounding box normalization to standard coordinate spaces like PyTorch's [-1, 1] [S1, unofficial]. It provides a unified interface for sampling voxel grid values at both integer coordinates (ijk) and floating-point coordinates (xyz) with configurable polynomial orders [S2, unofficial]. Unlike its sibling [Voxel Trilinear Interpolation (SIMD)](2564-voxel-trilinear-interpolation-simd.md), which focuses on SIMD-optimized trilinear sampling with boundary checks, this method supports higher-order interpolation (up to tricubic) and integrates with various data structures including dense grids and sparse voxel trees (OpenVDB/NanoVDB) [S2, unofficial][S3].

## How it works
**Inputs and Outputs:**
- **Input Volume:** A 3D grid of values (dense array or sparse tree).
- **Query Points:** A set of 3D coordinates $(x, y, z)$, either integer indices or floating-point positions.
- **Interpolation Order:** An integer specifying the method: 0 (Nearest), 1 (Trilinear), 2 (Triquadratic), 3 (Tricubic) [S2, unofficial].
- **Output:** The interpolated scalar value at each query point.

**Data Structures:**
- **Dense Grid:** Typically stored as a 5D tensor for batch processing: $(1, 1, D, H, W)$ where $D$ is depth, $H$ is height, $W$ is width [S1, unofficial].
- **Query Grid:** A tensor of shape $(1, N, 3)$ containing $N$ query points in normalized coordinates [S1, unofficial].
- **Sparse Tree:** For OpenVDB/NanoVDB, a tree structure with nodes and values, accessed via an accessor [S3].

**Steps:**

1.  **Coordinate Normalization (for Dense Grids):**
    If the query points are defined in world space or a specific bounding box, they must be normalized to the $[-1, 1]$ range expected by libraries like PyTorch [S1, unofficial].
    - Calculate the center of the bounding box:
      $$ \text{center} = \frac{\text{bbox}_{\text{min}} + \text{bbox}_{\text{max}}}{2} $$
    - Calculate the extent (half-width) of the bounding box:
      $$ \text{extent} = \frac{\text{bbox}_{\text{max}} - \text{bbox}_{\text{min}}}{2} $$
    - Normalize the query points $P$:
      $$ P_{\text{norm}} = \frac{P - \text{center}}{\text{extent}} $$
    [S1, unofficial]

2.  **Grid Generation (for Dense Grids):**
    - Generate meshgrid coordinates along the normalized axes using linear spacing (e.g., `torch.linspace`) [S1, unofficial].
    - Handle axis ordering: Reverse axis order to map XZY to ZXY, then stack and reverse again to XYZ to match the volume layout [S1, unofficial].
    - Reshape the input volume to $(1, 1, D, H, W)$ and the grid to $(1, N, 3)$ [S1, unofficial].

3.  **Sampling (Dense Grids):**
    - Call the interpolation function (e.g., `F.grid_sample`) with the input volume and the normalized grid [S1, unofficial].
    - Squeeze the output to remove batch and channel dimensions, resulting in a tensor of shape $(N)$ or $(N, 1)$ [S1, unofficial].

4.  **Sampling (Sparse Grids/OpenVDB):**
    - Create a sampler object using a factory function, specifying the interpolation order and accessor [S3]:
      ```python
      # Pseudocode for Sparse Sampling
      acc = grid.get_accessor()
      sampler = create_sampler(order=1, accessor=acc) # Order 1 for Trilinear
      ```
    [S3]
    - For each query point $P_i$:
      - If using integer coordinates, call the sampler with `CoordT ijk` [S2, unofficial].
      - If using floating-point coordinates, call the sampler with `Vec3f xyz` or `Vec3d xyz` [S2, unofficial].
      - The sampler traverses the tree to find the active voxel and computes the interpolated value [S2, unofficial][S3].
    - For Signed Distance Fields (SDF), use a specific sampler (e.g., `BoxSampler` for trilinear, `QuadraticSampler` for quadratic, `PointSampler` for nearest) and call `wsSample` [S1, unofficial]:
      ```python
      # Pseudocode for SDF Sampling
      sampler = GridSampler(accessor, transform, mode=1) # 1: BoxSampler (Trilinear)
      sdf_value = sampler.ws_sample(P_i)
      ```
    [S1, unofficial]

**Formulas:**
- **Trilinear Interpolation:**
  For a point $(x, y, z)$ within a voxel defined by corners $(i, j, k)$ to $(i+1, j+1, k+1)$, let $u, v, w$ be the fractional parts of $x, y, z$. The value $V$ is:
  $$ V = \sum_{a=0}^{1} \sum_{b=0}^{1} \sum_{c=0}^{1} w_a w_b w_c V_{i+a, j+b, k+c} $$
  where $w_0 = 1-u, w_1 = u$ (and similarly for $v, w$).
  [G: Standard trilinear formula, implied by "trilinear" in sources.]

- **Triquadratic Interpolation:**
  Uses 27 control points (3x3x3 neighborhood) with quadratic basis functions.
  [G: Implied by "Triquadratic" in sources.]

- **Tricubic Interpolation:**
  Uses 64 control points (4x4x4 neighborhood) with cubic basis functions.
  [G: Implied by "Tricubic" in sources.]

**Parameters:**
- **Order:** 0 (Nearest), 1 (Trilinear), 2 (Triquadratic), 3 (Tricubic) [S2, unofficial].
- **Mode:** 0 (Nearest), 1 (Trilinear), 2 (Quadratic) [S1, unofficial]. Note: Source [S1, unofficial] lists mode 2 as quadratic, while Source [S2, unofficial] lists order 2 as triquadratic and 3 as tricubic.
- **UseCache:** Boolean flag, defaults to `true` for sparse samplers [S3].

**Edge Cases and Pitfalls:**
- **Boundary Checks:** For dense grids, points outside the $[-1, 1]$ range may return padding values (e.g., 0) depending on the library settings [S1, unofficial]. For sparse grids, points outside the active volume may return undefined or default values; boundary checks are required [S2, unofficial].
- **Precision:** Floating-point coordinates should be handled with double precision (`Vec3d`) for high-accuracy applications, while single precision (`Vec3f`) may suffice for real-time rendering [S2, unofficial].
- **Axis Ordering:** Incorrect axis ordering (XYZ vs ZXY) can lead to misaligned sampling, especially when converting between different volume formats (e.g., VTK to PyTorch) [S1, unofficial].

## Subtopics

### [Tricubic B-Spline Interpolation](8246-tricubic-b-spline-interpolation.md)
A GPU-accelerated interpolation method using uniform cubic B-splines to sample a 3D volume texture, providing smoother results than linear interpolation while being faster than full tricubic convolution. It sets itself apart from standard tricubic convolution by using a specific optimized weight grouping and coordinate transformation that reduces the number of texture fetches and arithmetic operations, making it suitable for real-time volume rendering on WebGL-compatible hardware.

### Voxel Sampler Factory
A factory function that creates a voxel sampler of a specific polynomial order for sparse grids (OpenVDB/NanoVDB) [S3]. It allows for compact syntax and supports caching [S3].
- **Order:** Polynomial order for the sampler (e.g., 1 for trilinear) [S3].
- **UseCache:** Boolean flag, defaults to `true` [S3].
- **Usage:** `auto smp = nanovdb::math::createSampler<1>(acc);` [S3].

### SDF Grid Sampling
Evaluates the signed distance field at arbitrary query points using different interpolation strategies [S1, unofficial]. It uses specific sampler types: `BoxSampler` (trilinear), `QuadraticSampler` (quadratic), and `PointSampler` (nearest neighbor) [S1, unofficial].
- **Mode:** 1 (BoxSampler/Trilinear), 2 (QuadraticSampler), else (PointSampler/Nearest) [S1, unofficial].
- **Usage:** `sdf[i] = sampler_local.wsSample(openvdb::Vec3d(P(i, 0), P(i, 1), P(i, 2)));` [S1, unofficial].

## Operating characteristics
- **Time Complexity:**
  - Nearest Neighbor: $O(1)$ per query [G].
  - Trilinear: $O(1)$ per query, involving 8 texture fetches and 7 additions/multiplications [G].
  - Triquadratic: $O(1)$ per query, involving 27 texture fetches [G].
  - Tricubic: $O(1)$ per query, involving 64 texture fetches [G].
  - Sparse Grids: Depends on tree depth; typically $O(\log N)$ for traversal plus interpolation cost [G].
- **Memory Complexity:**
  - Dense Grids: $O(D \times H \times W)$ for the volume data [S1, unofficial].
  - Sparse Grids: Proportional to the number of active voxels and tree structure overhead [S3].
- **Measured Performance:**
  - Not given in the sources.
- **Scaling:**
  - Dense grids scale linearly with resolution [S1, unofficial].
  - Sparse grids scale with the complexity of the active volume [S3].
- **Strengths:**
  - Supports multiple interpolation orders [S2, unofficial].
  - Integrates with standard deep learning frameworks (PyTorch) [S1, unofficial].
  - Efficient for sparse data using OpenVDB/NanoVDB [S3].
- **Limitations:**
  - Higher-order interpolation (Triquadratic, Tricubic) is computationally expensive [S1, unofficial].
  - Boundary handling requires explicit normalization [S1, unofficial].
- **When to Use:**
  - Use Nearest for speed and discrete data [S1, unofficial].
  - Use Trilinear for a balance of speed and smoothness [S1, unofficial].
  - Use Triquadratic/Tricubic for high-quality smoothness in offline rendering [S1, unofficial].
  - Use Sparse Grids for large, sparse volumes [S3].

## Background
Voxel interpolation is a fundamental technique in volume rendering and scientific visualization. The integration with PyTorch's `F.grid_sample` allows for differentiable volume sampling in deep learning applications [S1, unofficial]. The use of OpenVDB and NanoVDB for sparse voxel interpolation enables efficient handling of large-scale volumetric data, such as in VFX and medical imaging [S2, unofficial][S3]. The method has evolved to support various interpolation orders to balance computational cost and visual quality [S2, unofficial].

## Sources
[S1] zzilch/Mesh2Volume, https://github.com/zzilch/Mesh2Volume, fetched 2026-10-26 (unofficial)
[S2] ZephirFXEC/HNanoSolver, https://github.com/ZephirFXEC/HNanoSolver, fetched 2026-10-27 (unofficial)
[S3] OpenVDB: nanovdb::math Namespace Reference, https://openvdb.org/documentation/doxygen/namespacenanovdb_1_1math.html, fetched 2026-10-24
