# Coordinate Space Conversion

Also called: Voxel Coordinate Conversion, Voxel Coordinate Mapping, Voxel Space Mapping, Voxel/Block/World coordinate conversions

## Overview

Coordinate Space Conversion defines the deterministic arithmetic for mapping between continuous world coordinates, continuous voxel coordinates, discrete integer voxel indices, and block indices in a sparse voxel grid. It solves the problem of accurately locating which voxel contains a given point and converting that voxel back to world space, ensuring consistency across host (CPU), device (GPU), and shader (GLSL) execution paths. It differs from [Voxel Grid Initialization](2168-voxel-grid-initialization.md) by performing the per-point arithmetic rather than static bounding box setup, and from [World to UVW Normalization](2229-world-to-uvw-normalization.md) by operating in physical units and integer indices rather than normalized [0, 1] texture coordinates.

## How it works

### Inputs and Outputs

*   **Inputs**:
    *   `world_pos`: A 3D floating-point vector $(x, y, z)$ in world units (e.g., meters).
    *   `voxel_size`: The physical size of a single voxel (scalar).
    *   `block_size`: The number of voxels along one axis of a block (scalar).
    *   `bounds`: Optional axis-aligned bounding box $[lb, ub]$ defining the valid grid extent.
    *   `dims`: The dimensions of the grid $[dim_x, dim_y, dim_z]$ in voxels.
*   **Outputs**:
    *   `voxel_idx`: A 3D integer vector $(i, j, k)$ representing the discrete voxel index.
    *   `block_idx`: A 3D integer vector representing the block index.
    *   `world_center`: The world-space center coordinate of a specified voxel.

### Data Structures

The method relies on the following parameters, typically stored in a `VoxelGridParams` structure:

*   `voxel_size` (float): Physical size of one voxel.
*   `block_size` (int): Number of voxels per block dimension.
*   `lb` (Vec3f): Lower bound of the grid in world space.
*   `ub` (Vec3f): Upper bound of the grid in world space.
*   `size` (Vec3i): Grid dimensions in voxels.
*   `scale` (Vec3f): Precomputed scaling factor, typically $scale = \frac{ub - lb}{size}$ or similar inverse mapping factor depending on the specific implementation variant [S2, S5, unofficial].

### Steps and Formulas

The conversion pipeline consists of three main transformations: World to Continuous Voxel, Continuous Voxel to Discrete Integer Voxel, and Integer Voxel to Block.

#### 1. World to Continuous Voxel Coordinate

Convert the world position to a continuous coordinate in voxel space.

$$
v = \frac{world\_pos}{voxel\_size}
$$

If a specific grid offset or lower bound $lb$ is used instead of a pure scale:

$$
v = \frac{world\_pos - lb}{voxel\_size}
$$

Where $v = (v_x, v_y, v_z)$ [S5, unofficial].

#### 2. Continuous Voxel to Discrete Integer Voxel

Map the continuous coordinate $v$ to the integer index $i$ of the voxel containing it. This step requires careful handling of rounding to ensure consistency across different hardware floating-point modes.

**Standard Floor Method (for containment):**
To find the voxel containing a point, use the floor function:

$$
i = \lfloor v \rfloor
$$

Specifically, for each axis $a \in \{x, y, z\}$:

$$
i_a = \lfloor v_a \rfloor
$$

This is implemented as `static_cast<int>(std::floor(v_a))` [S3, unofficial].

**Nearest Neighbor Rounding (for center approximation):**
If the goal is to find the nearest voxel center (e.g., for sampling), use **round-half-to-even** (ties to even). Standard functions like `std::rint` or `std::nearbyint` are **unsafe** because they respect the dynamic floating-point rounding mode (`fesetround`), which may differ between CPU and GPU/GLSL paths [S5, unofficial].

Instead, derive ties-to-even from `std::floor`:

1.  Compute lower integer: $lower = \lfloor v_a \rfloor$
2.  Compute fractional part: $frac = v_a - lower$
3.  Apply logic:
    *   If $frac < 0.5$, return $lower$.
    *   If $frac > 0.5$, return $lower + 1$.
    *   If $frac = 0.5$, return $lower$ if $\text{fmod}(lower, 2.0) = 0.0$, else $lower + 1$.

Pseudocode for nearest voxel:

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def round_half_to_even(v: float) -> int:
    lower = floor(v)
    frac = v - lower
    if frac < 0.5:
        return lower
    elif frac > 0.5:
        return lower + 1
    else: # frac == 0.5
        if fmod(lower, 2.0) == 0.0:
            return lower
        else:
            return lower + 1

def world_to_nearest_voxel(world_pos: Vec3, voxel_size: float) -> Vec3i:
    # Continuous voxel coordinate
    v = world_pos / voxel_size
    return Vec3i(round_half_to_even(v.x), round_half_to_even(v.y), round_half_to_even(v.z))
```

**Clamping:**
If the grid has a defined bounding box, clamp the resulting integer index $i$ to the valid range $[0, dims[axis] - 1]$ (or $[0, dims[axis] - 2]$ if the upper bound is exclusive or handled differently by the specific storage layout) [S4, unofficial].

#### 3. Discrete Integer Voxel to Block Index

Convert the integer voxel index to a block index. This requires **flooring toward negative infinity**, not truncation toward zero, because C++ integer division truncates toward zero [S5, unofficial].

**Algorithm:**
1.  If any component of the voxel index is negative, bias it by `block_size - 1` before division.
2.  Perform integer division by `block_size`.

$$
block\_idx = \left\lfloor \frac{voxel\_idx}{block\_size} \right\rfloor
$$

Implementation detail:

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_to_block(voxel: Vec3i, block_size: int) -> Vec3i:
    result = Vec3i()
    for axis in [0, 1, 2]:
        v = voxel[axis]
        if v < 0:
            v -= block_size - 1
        result[axis] = v // block_size
    return result
```

#### 4. Discrete Integer Voxel to World Center

Convert an integer voxel index back to the world-space center of that voxel.

$$
world\_center = (voxel\_idx + 0.5) \times voxel\_size
$$

Or, if using a lower bound $lb$:

$$
world\_center = lb + (voxel\_idx + 0.5) \times voxel\_size
$$

This assumes the center of voxel $[0,0,0]$ is at floating-point coordinate $[0.5, 0.5, 0.5]$ in continuous voxel space [S3, unofficial].

### Parameters

*   `voxel_size`: Physical size of a single voxel in meters. Typical values depend on the scene scale (e.g., 0.01m for detailed indoor, 1.0m for large outdoor) [S5, unofficial].
*   `block_size`: Number of voxels per block dimension. Common powers of 2 (e.g., 8, 16) are used for efficient bit-shifting in sparse implementations [S5, unofficial].
*   `trunc_dist`: TSDF truncation distance in meters. Used to compute the allocation band half-width in blocks: $blocks = \max(1, \lceil \frac{trunc\_dist}{block\_size \times voxel\_size} \rceil)$ [S5, unofficial].

### Edge Cases and Pitfalls

1.  **Rounding Mode Mismatch**: Using `std::rint` or hardware-native rounding instructions can cause discrepancies between CPU and GPU/GLSL code if the floating-point rounding mode is changed on the host. Always use the `floor`-based tie-to-even implementation [S5, unofficial].
2.  **Negative Index Division**: C++ `/` operator truncates toward zero. For negative voxel coordinates (e.g., voxel -1), integer division by block size (e.g., 8) yields 0, which is incorrect (voxel -1 belongs to block -1). The bias step (`v -= block_size - 1`) is mandatory for negative indices [S5, unofficial].
3.  **Boundary Clamping**: Points outside the defined grid bounds must be clamped to the nearest valid voxel index to prevent out-of-bounds memory access. The valid range is typically $[0, dims - 1]$ [S4, unofficial].
4.  **Center vs. Corner Convention**: Ensure consistency between the coordinate system used for indexing (lower corner) and the sampling point (center). The center is always offset by $0.5$ voxels from the integer index [S3, unofficial].

## Subtopics

### [Voxel Space Setup](3520-voxel-space-setup.md)
Computes the bounding box and transformation matrices required to map continuous world coordinates to discrete voxel grid indices, handling padding and cube voxel normalization. It differs from this article by performing the static initialization of the bounding volume and matrices, whereas this article performs the per-point arithmetic.

### [Voxel Grid Value Location Convention](13747-voxel-grid-value-location-convention.md)
Defines a coordinate system for dense volume grids where scalar field values are stored at the **lower corners** (minima) of voxels rather than at their centers, aligning with the OpenVDB sampling model. It eliminates the need for manual 0.5 voxel offsets during grid-to-mesh conversion.

## Operating characteristics

*   **Time Complexity**: $O(1)$ per coordinate conversion. The operations involve a constant number of floating-point multiplications, divisions, and integer arithmetic operations.
*   **Memory Complexity**: $O(1)$ auxiliary memory. Only a few scalar variables are needed per conversion.
*   **Strengths**: Deterministic and consistent across different execution environments (CPU/GPU/Shader) when the tie-to-even rounding is implemented correctly. Efficient for sparse voxel grids where block indexing is required.
*   **Limitations**: Requires careful handling of negative coordinates and floating-point rounding modes. Performance is not a bottleneck compared to memory access or ray marching, but incorrect implementation leads to subtle visual artifacts or crashes.
*   **When to use**: Whenever mapping between world space and voxel space is required, such as during ray traversal, TSDF fusion, or sparse grid traversal.

## Background

The concept of coordinate space conversion in voxel grids is fundamental to production volume rendering. It is discussed in the context of defining local, voxel, and world spaces in "Production Volume Rendering" [S3, unofficial], where the center of voxel $[0,0,0]$ is defined at $[0.5, 0.5, 0.5]$ in floating-point coordinates. The specific implementation details for sparse voxel grids, including block indexing and rounding consistency, are detailed in the `volumetric_kit_grid` repository [S5, unofficial]. The general principle of mapping continuous to discrete coordinates is also covered in "What Are The Coordinates Of A Pixel?" by Paul S. Heckbert [S3, unofficial].

## Sources

[S1] vinceniko/Isosurface, https://github.com/vinceniko/Isosurface, fetched 2026-10-27 (unofficial)
[S2] topskychen/voxelizer, https://github.com/topskychen/voxelizer, fetched 2026-10-27 (unofficial)
[S3] Production Volume Rendering, https://cg.informatik.uni-freiburg.de/intern/seminar/volumeRenderingProductionFundamentals2011.pdf, fetched 2026-10-27 (unofficial)
[S4] SRaimondi/VolumeRender, https://github.com/SRaimondi/VolumeRender, fetched 2026-10-27 (unofficial)
[S5] taojin-6/volumetric_kit_grid, https://github.com/taojin-6/volumetric_kit_grid, fetched 2026-10-28 (unofficial)
