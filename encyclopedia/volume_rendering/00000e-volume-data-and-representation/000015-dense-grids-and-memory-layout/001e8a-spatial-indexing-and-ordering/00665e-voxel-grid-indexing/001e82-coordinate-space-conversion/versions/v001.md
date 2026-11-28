# Coordinate Space Conversion

Also called: Voxel Coordinate Conversion, Discrete/Continuous Conversion, Sparse Voxel Grid Coordinate Transforms, Voxel Coordinate Mapping, Voxel Space Mapping, Voxel/Block/World coordinate conversions

## Overview
Coordinate Space Conversion defines the deterministic mappings between world-space floating-point positions, continuous voxel coordinates, integer voxel indices, and block indices in a sparse voxel grid [S4, unofficial]. It solves the problem of accurately locating which voxel contains a given point and translating between the discrete indexing required for memory access and the continuous coordinates required for interpolation or rendering [S2, unofficial]. It differs from [World to UVW Normalization](2229-world-to-uvw-normalization.md) by operating on physical units and integer grid topology rather than normalized texture coordinates, and from [Bitstream Voxel Position Encoding](2371-bitstream-voxel-position-encoding.md) by providing the arithmetic transforms that precede or replace bit-packing [S4, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D point in world space ($P_{world}$), grid parameters (voxel size, block size, bounds), and optionally a continuous voxel coordinate.
- Output: Integer voxel index ($V_{idx}$), continuous voxel coordinate ($V_{cont}$), block index ($B_{idx}$), or world-space center.

**Data Structures:**
The conversion relies on the following grid parameters, typically stored in a `VoxelGridParams` structure [S4, unofficial]:
- `voxel_size`: Physical size of a single voxel in meters [S4, unofficial].
- `block_size`: Number of voxels along one axis of a block (e.g., 8 or 16) [S4, unofficial].
- `bounds`: Axis-aligned bounding box (AABB) defining the valid range of the grid, often derived from mesh extents [S1, unofficial].
- `size_`: Grid dimensions in voxels ($N_x, N_y, N_z$) [S1, unofficial].
- `lb_`: Lower bound of the grid in world space [S1, unofficial].
- `scale_`: Precomputed scaling factor, typically $bound\_size / size_$ [S1, unofficial].

**Conversion Steps:**

1. **World to Continuous Voxel Coordinate:**
   Convert world position to continuous voxel space by dividing by voxel size.
   $$ V_{cont} = \frac{P_{world}}{voxel\_size} $$
   [S4, unofficial]

2. **Continuous Voxel to Integer Voxel Index (Rounding):**
   Map the continuous coordinate to the nearest integer voxel index. To ensure byte-identical results across host (CPU), device (CUDA), and shader (GLSL) paths regardless of the floating-point rounding mode, a custom "round half to even" (ties-to-even) function is required [S4, unofficial]. Standard functions like `std::rint` or `std::nearbyint` respect the dynamic rounding mode (`fesetround`), which can cause divergence [S4, unofficial].

   The implementation uses `std::floor` to derive the rounding logic [S4, unofficial]:
   ```python
   def round_half_to_even(x):
       lower = floor(x)
       frac = x - lower
       if frac < 0.5:
           return lower
       elif frac > 0.5:
           return lower + 1.0
       else: # frac == 0.5
           if fmod(lower, 2.0) == 0.0:
               return lower
           else:
               return lower + 1.0
   ```
   [S4, unofficial]

   Apply this to each component ($x, y, z$) of $V_{cont}$ and cast to `int` to get $V_{idx}$ [S4, unofficial].

   *Alternative Discrete/Continuous Mapping:*
   In simpler local-space contexts, discrete to continuous conversion assumes the center of voxel $i$ is at $i + 0.5$, and continuous to discrete uses floor [S2, unofficial]:
   $$ i = \lfloor c \rfloor $$
   $$ c = i + 0.5 $$
   [S2, unofficial]

3. **Integer Voxel to Block Index:**
   Convert integer voxel coordinates to block coordinates. Because C++ integer division truncates toward zero, but block indexing requires flooring toward negative infinity, negative voxel coordinates must be biased [S4, unofficial].
   
   For each axis:
   ```python
   if voxel_coord < 0:
       voxel_coord -= (block_size - 1)
   block_coord = voxel_coord // block_size
   ```
   [S4, unofficial]
   
   This ensures that voxel $-1$ (which belongs to block $-1$, spanning voxels $-block\_size \dots -1$) maps to block $-1$ instead of $0$ [S4, unofficial].

4. **Block to World Center:**
   Compose `block_to_voxel` and `voxel_to_world`. The result is the world-space center of the block's minimum-index voxel, not the block's geometric minimum corner [S4, unofficial].
   $$ V_{origin} = B_{idx} \times block\_size $$
   $$ P_{world\_center} = V_{origin} \times voxel\_size $$
   [S4, unofficial]

5. **World to Block:**
   Compose `world_to_voxel` (using nearest voxel rounding) and `voxel_to_block` [S4, unofficial].

6. **Index to Voxel (Linear to 3D):**
   For a linear array index $I$, recover the 3D voxel coordinates $(x, y, z)$ using precomputed strides [S1, unofficial]:
   $$ x = I / size\_yz\_ $$
   $$ y = (I / size\_z\_) \% size\_y\_ $$
   $$ z = I \% size\_z\_ $$
   Where $size\_yz\_ = size\_x\_ \times size\_y\_$ and $size\_xz\_ = size\_x\_ \times size\_z\_$ [S1, unofficial].

7. **Voxel to Index (3D to Linear):**
   Convert 3D voxel coordinates back to a linear index using the `INDEX` macro or equivalent stride multiplication [S1, unofficial].

**Parameters:**
- `voxel_size`: Grid parameter [S4, unofficial].
- `block_size`: Grid parameter [S4, unofficial].
- `trunc_dist`: TSDF truncation distance in meters. Used to compute the allocation band half-width in blocks: $blocks = \max(1, \lceil \frac{trunc\_dist}{block\_size \times voxel\_size} \rceil)$ [S4, unofficial].
- `size_yz_`, `size_xz_`: Precomputed strides [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Rounding Mode Divergence:** Using standard library rounding functions that respect `fesetround` causes host and device code to round differently, assigning samples to the wrong voxel/block [S4, unofficial].
- **Negative Index Division:** Failing to bias negative voxel coordinates before integer division results in incorrect block mapping (e.g., voxel $-1$ mapping to block $0$) [S4, unofficial].
- **Boundary Clamping:** When mapping world points to voxel indices, the result must be clamped to the valid range $[0, dims[axis] - 1]$ (or $-2$ depending on specific implementation details, typically $0$ to $N-1$) [S3, unofficial]. Note: Source [S3, unofficial] specifies clamping to `[0, dims[axis] - 2]`, which may imply a specific boundary handling or off-by-one convention in that implementation; standard practice is $[0, N-1]$ [S3, unofficial].
- **Local Space Range:** Local space maps the full extents of the voxel buffer to a $[0, 1]$ range, providing a resolution-independent coordinate system [S2, unofficial].

## Subtopics

### [Voxel Space Setup](3520-voxel-space-setup.md)
Computes bounding boxes and transformation matrices for mapping world coordinates to voxel grid indices, handling padding and cube voxel normalization. It explicitly manages discretization extent and generates projection matrices for NDC suitable for rasterization-based voxelization.

## Operating characteristics
- **Time Complexity:** $O(1)$ per coordinate conversion.
- **Memory Complexity:** $O(1)$ auxiliary storage for precomputed strides and bounds.
- **Strengths:** Deterministic behavior across hardware platforms (CPU/GPU/Shader) due to explicit tie-to-even rounding; handles negative coordinates correctly for sparse grids.
- **Limitations:** Requires careful management of grid parameters (bounds, sizes) to avoid out-of-bounds access.
- **When to use:** Whenever world-space points must be mapped to discrete voxel indices for storage, traversal, or rendering in a sparse voxel grid.
- **When not to use:** For boundless fields where local space concepts do not apply, only world-to-voxel transforms are needed [S2, unofficial].

## Background
The concept of coordinate space conversion in voxel rendering is foundational, with early discussions on pixel/voxel coordinates by Paul S. Heckbert [Heckbert, 1990] cited in [S2, unofficial]. The specific implementation details regarding tie-to-even rounding and block indexing biases are documented in recent sparse voxel grid implementations [S4, unofficial]. The linear indexing strategies are found in collision detection voxelizers [S1, unofficial].

## Sources
[S1] topskychen/voxelizer, https://github.com/topskychen/voxelizer, fetched 2026-10-27 (unofficial)
[S2] Production Volume Rendering, https://cg.informatik.uni-freiburg.de/intern/seminar/volumeRenderingProductionFundamentals2011.pdf, fetched 2026-10-27 (unofficial)
[S3] SRaimondi/VolumeRender, https://github.com/SRaimondi/VolumeRender, fetched 2026-10-27 (unofficial)
[S4] taojin-6/volumetric_kit_grid, https://github.com/taojin-6/volumetric_kit_grid, fetched 2026-10-28 (unofficial)
