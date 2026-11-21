# Coordinate Space Conversion

Also called: Voxel Coordinate Conversion, Voxel Coordinate Mapping, Voxel Space Mapping, Voxel/Block/World coordinate conversions

## Overview
Coordinate Space Conversion defines the deterministic mappings between world-space floating-point positions, continuous voxel coordinates, integer voxel indices, and block indices in a sparse voxel grid [S4, unofficial]. It solves the problem of accurately locating which voxel contains a given point and translating between the discrete indexing required for memory access and the continuous coordinates required for interpolation or rendering [S2, unofficial]. It differs from [World to UVW Normalization](2229-world-to-uvw-normalization.md) by operating on physical units and integer grid topology rather than normalized texture coordinates, and from [Bitstream Voxel Position Encoding](2371-bitstream-voxel-position-encoding.md) by providing the arithmetic transforms that precede or replace bit-packing [S4, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D point in world space ($P_{world}$), grid parameters (voxel size, block size, bounds), and optionally a continuous voxel coordinate [S4, unofficial].
- Output: Integer voxel index ($V_{idx}$), continuous voxel coordinate ($V_{cont}$), block index ($B_{idx}$), or world-space center [S5, unofficial].

**Data Structures:**
The conversion relies on the following grid parameters, typically stored in a `VoxelGridParams` structure [S4, unofficial]:
- `voxel_size`: Physical size of a single voxel in meters [S5, unofficial].
- `block_size`: Number of voxels along one axis of a block [S5, unofficial].
- `bounds`: Axis-aligned bounding box (AABB) defining the valid range of the grid [S1, unofficial].
- `size_`: Grid dimensions in voxels ($N_x, N_y, N_z$) [S1, unofficial].
- `lb_`: Lower bound of the grid in world space [S2, unofficial].
- `scale_`: Precomputed scaling factor, typically $bound\_size / size_$ [S2, unofficial].

**Conversion Steps:**

1. **World to Continuous Voxel Coordinate:**
   Convert world position to continuous voxel space by dividing by voxel size [S5, unofficial]:
   $$ V_{cont} = \frac{P_{world}}{voxel\_size} $$

2. **Continuous Voxel to Integer Voxel Index (Rounding):**
   Map the continuous coordinate to the nearest integer voxel index [S4, unofficial]. To ensure byte-identical results across host (CPU), device (CUDA), and shader (GLSL) paths regardless of the floating-point rounding mode, a custom "round half to even" (ties-to-even) function is required [S5, unofficial]. Standard functions like `std::rint` or `std::nearbyint` respect the dynamic rounding mode (`fesetround`), which causes divergence between host and device code [S5, unofficial].

   The implementation derives the rounding logic from `floor`, which ignores the dynamic mode [S5, unofficial]:
   ```python
   import math

   def round_half_to_even(x):
       lower = math.floor(x)
       frac = x - lower
       if frac < 0.5:
           return lower
       elif frac > 0.5:
           return lower + 1.0
       else:  # frac == 0.5
           if math.fmod(lower, 2.0) == 0.0:
               return lower
           else:
               return lower + 1.0
   ```
   Apply this to each component ($x, y, z$) of $V_{cont}$ and cast to `int` to get $V_{idx}$ [S5, unofficial].

   *Alternative Discrete/Continuous Mapping:*
   In simpler local-space contexts, discrete to continuous conversion assumes the center of voxel $i$ is at $i + 0.5$, and continuous to discrete uses floor [S3, unofficial]:
   $$ i = \lfloor c \rfloor $$
   $$ c = i + 0.5 $$
   [S3, unofficial]

3. **Integer Voxel to Block Index:**
   Convert integer voxel coordinates to block coordinates [S5, unofficial]. Because C++ integer division truncates toward zero, but block indexing requires flooring toward negative infinity, negative voxel coordinates must be biased [S5, unofficial].

   For each axis:
   ```python
   if voxel_coord < 0:
       voxel_coord -= (block_size - 1)
   block_coord = voxel_coord // block_size
   ```
   [S5, unofficial]
   This ensures that voxel $-1$ (which belongs to block $-1$, spanning voxels $-block\_size \dots -1$) maps to block $-1$ instead of $0$ [S5, unofficial].

4. **Block to World Center:**
   Compose `block_to_voxel` and `voxel_to_world` [S5, unofficial]. The result is the world-space center of the block's minimum-index voxel, not the block's geometric minimum corner [S5, unofficial]:
   $$ V_{origin} = B_{idx} \times block\_size $$
   $$ P_{world\_center} = V_{origin} \times voxel\_size $$
   [S5, unofficial]

5. **World to Block:**
   Compose `world_to_voxel` (using nearest voxel rounding) and `voxel_to_block` [S5, unofficial].

6. **Index to Voxel (Linear to 3D):**
   For a linear array index $I$, recover the 3D voxel coordinates $(x, y, z)$ using precomputed strides [S2, unofficial]:
   $$ x = I / size\_yz\_ $$
   $$ y = (I / size\_z\_) \% size\_y\_ $$
   $$ z = I \% size\_z\_ $$
   Where $size\_yz\_ = size\_x\_ \times size\_y\_$ and $size\_xz\_ = size\_x\_ \times size\_z\_$ [S2, unofficial].

7. **Voxel to Index (3D to Linear):**
   Convert 3D voxel coordinates back to a linear index using the `INDEX` macro or equivalent stride multiplication [S2, unofficial].

**Parameters:**
- `voxel_size`: Grid parameter [S5, unofficial].
- `block_size`: Grid parameter [S5, unofficial].
- `trunc_dist`: TSDF truncation distance in meters [S5, unofficial]. Used to compute the allocation band half-width in blocks: $blocks = \max(1, \lceil \frac{trunc\_dist}{block\_size \times voxel\_size} \rceil)$ [S5, unofficial].
- `size_yz_`, `size_xz_`: Precomputed strides [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Rounding Mode Divergence:** Using standard library rounding functions that respect `fesetround` causes host and device code to round differently, assigning samples to the wrong voxel/block [S5, unofficial].
- **Negative Index Division:** Failing to bias negative voxel coordinates before integer division results in incorrect block mapping (e.g., voxel $-1$ mapping to block $0$) [S5, unofficial].
- **Boundary Clamping:** When mapping world points to voxel indices, the result must be clamped to the valid range $[0, dims[axis] - 2]$ [S4, unofficial].
- **Local Space Range:** Local space maps the full extents of the voxel buffer to a $[0, 1]$ range, providing a resolution-independent coordinate system [S3, unofficial].
- **Voxel Center Offset:** The center of voxel $[0, 0, 0]$ has floating-point coordinates $[0.5, 0.5, 0.5]$, meaning edges of a field with resolution $N$ are at $0.0$ and $N.0$ in floating-point coordinates, while integer indices only span $0$ through $N-1$ [S3, unofficial].

## Subtopics

### [Voxel Space Setup](3520-voxel-space-setup.md)
Computes bounding boxes and transformation matrices for mapping world coordinates to voxel grid indices, handling padding and cube voxel normalization. It explicitly manages discretization extent and generates projection matrices for NDC suitable for rasterization-based voxelization.

## Operating characteristics
- **Time Complexity:** $O(1)$ per coordinate conversion [S4, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary storage for precomputed strides and bounds [S2, unofficial].
- **Strengths:** Deterministic behavior across hardware platforms (CPU/GPU/Shader) due to explicit tie-to-even rounding; handles negative coordinates correctly for sparse grids [S5, unofficial].
- **Limitations:** Requires careful management of grid parameters (bounds, sizes) to avoid out-of-bounds access [S4, unofficial].
- **When to use:** Whenever world-space points must be mapped to discrete voxel indices for storage, traversal, or rendering in a sparse voxel grid [S5, unofficial].
- **When not to use:** For boundless fields where local space concepts do not apply, only world-to-voxel transforms are needed [S3, unofficial].

## Background
The concept of coordinate space conversion in voxel rendering is foundational, with early discussions on pixel/voxel coordinates by Paul S. Heckbert [Heckbert, 1990] cited in [S3, unofficial]. The specific implementation details regarding tie-to-even rounding and block indexing biases are documented in recent sparse voxel grid implementations [S5, unofficial]. The linear indexing strategies are found in collision detection voxelizers [S2, unofficial].

## Sources
[S1] vinceniko/Isosurface, https://github.com/vinceniko/Isosurface, fetched 2026-10-27 (unofficial)
[S2] topskychen/voxelizer, https://github.com/topskychen/voxelizer, fetched 2026-10-27 (unofficial)
[S3] Production Volume Rendering, https://cg.informatik.uni-freiburg.de/intern/seminar/volumeRenderingProductionFundamentals2011.pdf, fetched 2026-10-27 (unofficial)
[S4] SRaimondi/VolumeRender, https://github.com/SRaimondi/VolumeRender, fetched 2026-10-27 (unofficial)
[S5] taojin-6/volumetric_kit_grid, https://github.com/taojin-6/volumetric_kit_grid, fetched 2026-10-28 (unofficial)
