# Volume Geometry Transformations

Also called: Spatial attribute modification, VolumeGeometry methods, multiply, reshape, translate, untranslate

## Overview
Volume Geometry Transformations modify the spatial attributes (position, size, shape) of a 3D volume geometry while preserving the underlying voxel grid topology, except for `reshape` which changes the grid dimensions [S1, unofficial]. It sets itself apart by operating directly on the `VolumeGeometry` metadata (shape, position, size) rather than resampling the voxel data itself, allowing for efficient coordinate system adjustments without data interpolation [S1, unofficial].

## How it works
**Inputs and Outputs:**
The methods operate on a `VolumeGeometry` object, which contains:
- `shape`: A tuple of integers `(nx, ny, nz)` representing the grid dimensions.
- `pos`: A tuple of tuples `((x0, y0, z0), ...)` or similar structure defining the coordinate positions of the grid corners/centers. The source code snippet `self.pos[0]` implies `pos` is an iterable of coordinate tuples.
- `size`: A tuple `(sx, sy, sz)` representing the physical dimensions of the volume.

The output is a new `VolumeGeometry` instance with updated attributes.

**Data Structures:**
The `VolumeGeometry` class holds the spatial metadata. The exact internal layout is not fully documented, but the methods access `self.shape`, `self.pos`, and `self.size` directly [S1, unofficial].

**Steps and Formulas:**

1.  **Translate (`translate(t)`)**
    Shifts the volume's position by a translation vector $t = (t_x, t_y, t_z)$.
    -   Formula: $p'_{i} = p_i + t_i$ for each coordinate component $i$ in the position tuple.
    -   Implementation:
        ```python
        new_pos = tuple(p + t for p, t in zip(self.pos[0], t))
        return VolumeGeometry(shape=self.shape, pos=new_pos, size=self.size)
        ```
    -   Parameters: `t` is a 3D vector.

2.  **Scale (`scale(scale)`)**
    Changes the physical size of the volume without moving its position.
    -   Formula: $s'_i = s_i \times k_i$ where $k$ is the scale factor.
    -   Implementation:
        ```python
        new_size = tuple(a * b for a, b in zip(scale, self.size))
        return VolumeGeometry(shape=self.shape, pos=self.pos[0], size=new_size)
        ```
    -   Parameters: `scale` is a 3D vector $(k_x, k_y, k_z)$.

3.  **Multiply (`multiply(scale)`)**
    Scales the volume's size and simultaneously scales its position coordinates relative to the origin.
    -   Formula:
        $$s'_i = s_i \times k_i$$
        $$p'_i = p_i \times k_i$$
    -   Implementation:
        ```python
        new_size = tuple(a * b for a, b in zip(scale, self.size))
        new_pos = tuple(a * b for a, b in zip(scale, self.pos[0]))
        return VolumeGeometry(shape=self.shape, pos=new_pos, size=new_size)
        ```
    -   Parameters: `scale` is a 3D vector.

4.  **With Voxel Size (`with_voxel_size(voxel_size)`)**
    Recalculates the grid shape to fit a target voxel dimension, centering the volume on the origin.
    -   Formula:
        $$n'_i = \lfloor s_i / v_i \rfloor$$
        $$s'_i = n'_i \times v_i$$
        The new position is adjusted so the volume is centered on the origin.
    -   Implementation:
        ```python
        voxel_size_3d = tuple(voxel_size) if not isinstance(voxel_size, tuple) else voxel_size
        new_shape = (np.array(self.size) / voxel_size_3d).astype(int)
        new_size = new_shape * voxel_size_3d
        # Note: The source states the result is centered on the origin.
        # The exact calculation of new_pos for centering is not explicitly coded in the snippet,
        # but implies pos is adjusted to reflect the new size centered at 0.
        return VolumeGeometry(new_shape, pos=self.pos[0], size=new_size)
        ```
    -   Parameters: `voxel_size` is a scalar or 3D vector.

5.  **Reshape (`reshape(new_shape)`)**
    Changes the grid dimensions while keeping the physical size and position constant.
    -   Implementation:
        ```python
        if isinstance(new_shape, int):
            new_shape = (new_shape, new_shape, new_shape)
        return VolumeGeometry(new_shape, pos=self.pos[0], size=self.size)
        ```
    -   Parameters: `new_shape` is an int or a 3-tuple of ints.

6.  **Right Multiply (`__rmul__(other)`)**
    Handles multiplication of a `Transform` object with the `VolumeGeometry`.
    -   If `other` is a `Transform` with `num_steps == 1`:
        1.  Extract translation: $t = \text{other.matrix}[0, :3, 3]$.
        2.  Extract scale: $k = |\text{other.matrix}[0].\text{diagonal}()[:3]|$.
        3.  Construct a composite transform $T = \text{translate}(t) \times \text{scale}(k)$.
        4.  If $T == \text{other}$, return `self.multiply(scale).translate(translation)`.
        5.  Otherwise, warn and convert to `VolumeVectorGeometry` via `other * self.to_vec()`.
    -   Pitfall: Using `abs()` on the diagonal masks negative scaling (reflections), potentially altering geometry orientation [S1, unofficial].

**Edge Cases and Pitfalls:**
-   **Negative Scaling:** The `__rmul__` method uses `abs()` on the matrix diagonal to extract scale because `ts.scale` throws an error on negative values. This hides reflection transforms, leading to incorrect orientation if the original transform included a flip [S1, unofficial].
-   **Voxel Size Rounding:** `with_voxel_size` rounds down the shape calculation. This may result in a volume that is slightly smaller than the original physical size, fitting inside the original bounds [S1, unofficial].
-   **Reshape Validity:** `reshape` changes the number of voxels but not the physical size. This implies the voxel size changes. If the downstream renderer expects a specific voxel size, this may cause artifacts.

## Operating characteristics
-   **Time Complexity:** $O(1)$ for `translate`, `scale`, `multiply`, `reshape`. $O(N)$ for `with_voxel_size` if array operations are considered, but effectively constant for typical 3D volumes.
-   **Memory Complexity:** $O(1)$ auxiliary memory, as it only creates a new metadata object.
-   **Strengths:** Fast, no data resampling required, preserves voxel topology (except reshape).
-   **Limitations:** Does not handle complex non-linear transforms. `__rmul__` fallback to `VolumeVectorGeometry` may be computationally heavier.
-   **When to use:** Adjusting coordinate systems, changing voxel resolution metadata, or applying simple affine transforms to the geometry definition.
-   **When not to use:** When the actual voxel data needs to be interpolated or rotated in space (use resampling methods instead).

## Background
The `VolumeGeometry` transformations are part of the TomoSipo library, a Python package for tomographic reconstruction [S1, unofficial]. The library provides a high-level interface for defining projection and volume geometries, allowing users to manipulate these geometries before creating operators for forward and backprojection [S1, unofficial]. The specific methods `translate`, `scale`, `multiply`, `with_voxel_size`, and `reshape` are designed to facilitate the setup of volume geometries for various tomographic applications [S1, unofficial].

## Sources
[S1] cicwi/tomosipo, https://github.com/cicwi/tomosipo, fetched 2026-10-26 (unofficial)
