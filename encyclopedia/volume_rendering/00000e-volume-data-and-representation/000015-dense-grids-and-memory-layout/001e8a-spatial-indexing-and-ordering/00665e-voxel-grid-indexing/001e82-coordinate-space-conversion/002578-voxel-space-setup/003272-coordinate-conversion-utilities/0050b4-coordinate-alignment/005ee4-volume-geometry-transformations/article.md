# Volume Geometry Transformations

Also called: Spatial attribute modification, VolumeGeometry methods, multiply, reshape, translate, untranslate

## Overview
Volume Geometry Transformations modify the spatial attributes (position, size, shape) of a 3D volume geometry while preserving the underlying voxel grid topology, except for `reshape` which changes the grid dimensions [S1, unofficial]. It sets itself apart by operating directly on the geometry metadata rather than resampling the voxel data, enabling efficient coordinate system adjustments without interpolation [S1, unofficial].

## How it works
**Inputs and Outputs:**
The methods accept a volume geometry object containing `shape` (a 3-tuple of integers), `pos` (an iterable of coordinate tuples, accessed as `pos[0]`), and `size` (a 3-tuple of physical dimensions), and return a new geometry object with updated attributes [S1, unofficial].

**Data Structures:**
The geometry object stores grid dimensions, origin coordinates, and physical extents [S1, unofficial].

**Steps and Formulas:**

1. **Translate (`translate(t)`)**
   Shifts the volume's position by a translation vector $t = (t_x, t_y, t_z)$ [S1, unofficial].
   - Formula: $p'_i = p_i + t_i$ for each coordinate component $i$ in the position tuple [S1, unofficial].
   - Implementation:
     ```python
     new_pos = tuple(p + ti for p, ti in zip(self.pos[0], t))
     return VolumeGeometry(shape=self.shape, pos=new_pos, size=self.size)
     ```

2. **Scale (`scale(scale)`)**
   Changes the physical size of the volume without moving its position [S1, unofficial].
   - Formula: $s'_i = s_i \times k_i$ where $k$ is the scale factor [S1, unofficial].
   - Implementation:
     ```python
     new_size = tuple(s * si for s, si in zip(self.size, scale))
     return VolumeGeometry(shape=self.shape, pos=self.pos[0], size=new_size)
     ```

3. **Multiply (`multiply(scale)`)**
   Scales the volume's size and simultaneously scales its position coordinates relative to the origin [S1, unofficial].
   - Formula: $s'_i = s_i \times k_i$ and $p'_i = p_i \times k_i$ [S1, unofficial].
   - Implementation:
     ```python
     new_size = tuple(s * si for s, si in zip(self.size, scale))
     new_pos = tuple(p * si for p, si in zip(self.pos[0], scale))
     return VolumeGeometry(shape=self.shape, pos=new_pos, size=new_size)
     ```

4. **With Voxel Size (`with_voxel_size(voxel_size)`)**
   Recalculates the grid shape to fit a target voxel dimension, centering the volume on the origin [S1, unofficial].
   - Formula: $n'_i = \lfloor s_i / v_i \rfloor$ and $s'_i = n'_i \times v_i$ [S1, unofficial].
   - Implementation:
     ```python
     vs = tuple(voxel_size) if not isinstance(voxel_size, tuple) else voxel_size
     new_shape = tuple(int(s / v) for s, v in zip(self.size, vs))
     new_size = tuple(n * v for n, v in zip(new_shape, vs))
     return VolumeGeometry(shape=new_shape, pos=self.pos[0], size=new_size)
     ```

5. **Reshape (`reshape(new_shape)`)**
   Changes the grid dimensions while keeping the physical size and position constant [S1, unofficial].
   - Implementation:
     ```python
     if isinstance(new_shape, int):
         new_shape = (new_shape, new_shape, new_shape)
     return VolumeGeometry(shape=new_shape, pos=self.pos[0], size=self.size)
     ```

6. **Right Multiply (`__rmul__(other)`)**
   Handles multiplication of a `Transform` object with the `VolumeGeometry` [S1, unofficial].
   - If `other` is a `Transform` with `num_steps == 1`:
     1. Extract translation: $t = \text{other.matrix}[0, :3, 3]$ [S1, unofficial].
     2. Extract scale: $k = |\text{other.matrix}[0].\text{diagonal}()[:3]|$ [S1, unofficial].
     3. Construct a composite transform $T = \text{translate}(t) \times \text{scale}(k)$ [S1, unofficial].
     4. If $T == \text{other}$, return `self.multiply(scale).translate(translation)` [S1, unofficial].
     5. Otherwise, warn and convert to `VolumeVectorGeometry` via `other * self.to_vec()` [S1, unofficial].
   - Pitfall: Using `abs()` on the diagonal masks negative scaling (reflections), potentially altering geometry orientation if the original transform included a flip [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Negative Scaling:** The `__rmul__` method uses `abs()` on the matrix diagonal to extract scale because `ts.scale` throws an error on negative values, hiding reflection transforms and leading to incorrect orientation [S1, unofficial].
- **Voxel Size Rounding:** `with_voxel_size` rounds down the shape calculation, resulting in a volume that fits inside the original bounds [S1, unofficial].
- **Reshape Validity:** `reshape` changes the number of voxels but not the physical size, implying the voxel size changes [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class VolumeGeometry:
    def __init__(self, shape, pos, size):
        self.shape = shape
        self.pos = pos
        self.size = size

    def translate(self, t):
        new_pos = tuple(p + ti for p, ti in zip(self.pos[0], t))
        return VolumeGeometry(shape=self.shape, pos=new_pos, size=self.size)

    def scale(self, scale):
        new_size = tuple(s * si for s, si in zip(self.size, scale))
        return VolumeGeometry(shape=self.shape, pos=self.pos[0], size=new_size)

    def multiply(self, scale):
        new_size = tuple(s * si for s, si in zip(self.size, scale))
        new_pos = tuple(p * si for p, si in zip(self.pos[0], scale))
        return VolumeGeometry(shape=self.shape, pos=new_pos, size=new_size)

    def with_voxel_size(self, voxel_size):
        vs = tuple(voxel_size) if not isinstance(voxel_size, tuple) else voxel_size
        new_shape = tuple(int(s / v) for s, v in zip(self.size, vs))
        new_size = tuple(n * v for n, v in zip(new_shape, vs))
        return VolumeGeometry(shape=new_shape, pos=self.pos[0], size=new_size)

    def reshape(self, new_shape):
        if isinstance(new_shape, int):
            new_shape = (new_shape, new_shape, new_shape)
        return VolumeGeometry(shape=new_shape, pos=self.pos[0], size=self.size)

    def __rmul__(self, other):
        if other.num_steps == 1:
            translation = other.matrix[0, :3, 3]
            scale = tuple(abs(d) for d in other.matrix[0].diagonal()[:3])
            # T = translate(translation) * scale(scale)
            if T == other:
                return self.multiply(scale).translate(translation)
            else:
                # Warn and convert to VolumeVectorGeometry
                return other * self.to_vec()
```

## Subtopics

### [Rotation](19376-rotation.md)
Rotation modifies the spatial orientation of a 3D volume geometry by applying angular transformations to voxel coordinates. It solves the problem of reorienting volumetric data for alignment or geometric manipulation while preserving the underlying voxel grid topology.

### Volume Unbending
Straightens a map near a path formed by atoms/bonds, mapping the path to the Z-axis of the result [S2]. Steps: fit a cubic spline through the path points, interpolate the input volume values on planes perpendicular to the splined path, map the path to the Z-axis, and map the existing volume's Y-axis to the new map's Y-axis [S2]. Parameters: `gridSpacing s` (default is the minimum spacing along the three axes of the input map), `xsize xs` and `ysize ys` (X and Y dimensions in physical units, default 10 times the new grid spacing) [S2].

### Axis Permutation
Rearranges the grid axes to a specified permutation order, supporting any of the 6 ordered combinations of x, y, and z, where the original order is xyz [S2].

## Operating characteristics
- **Time Complexity:** $O(1)$ for `translate`, `scale`, `multiply`, `reshape`, and $O(N)$ for `with_voxel_size` if array operations are considered, but effectively constant for typical 3D volumes [S1, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary memory, as it only creates a new metadata object without copying voxel data [S1, unofficial].
- **Strengths:** Fast execution, no data resampling required, preserves voxel topology (except `reshape`) [S1, unofficial].
- **Limitations:** Does not handle complex non-linear transforms; `__rmul__` fallback to `VolumeVectorGeometry` may be computationally heavier [S1, unofficial].
- **When to use:** Adjusting coordinate systems, changing voxel resolution metadata, or applying simple affine transforms to the geometry definition [S1, unofficial].
- **When not to use:** When the actual voxel data needs to be interpolated or rotated in space, requiring resampling methods instead [S1, unofficial].

## Background
The transformations originate from the TomoSipo library, a Python package for tomographic reconstruction [S1, unofficial]. The library provides a high-level interface for defining projection and volume geometries, allowing users to manipulate these geometries before creating operators for forward and backprojection [S1, unofficial]. The specific methods `translate`, `scale`, `multiply`, `with_voxel_size`, and `reshape` facilitate the setup of volume geometries for various tomographic applications [S1, unofficial]. Additional variants like `volume unbend` and `volume permuteAxes` are provided by the ChimeraX software for structural biology volume processing [S2].

## Sources
[S1] cicwi/tomosipo, https://github.com/cicwi/tomosipo, fetched 2026-10-26 (unofficial)
[S2] Command: volume, https://www.rbvi.ucsf.edu/chimerax/docs/user/commands/volume.html, fetched 2026-10-27
