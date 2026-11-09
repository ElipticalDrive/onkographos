# Cylindrical Coordinate Interpolation
Also called: Interpolation in cylindrical coordinates
## Overview
Cylindrical Coordinate Interpolation maps Cartesian query points to cylindrical space before sampling the volume data, enabling efficient texture-based lookup for geometries with cylindrical symmetry [S1, unofficial]. It differs from standard Cartesian interpolation by remapping the coordinate system to align with the geometry's principal axis [S1, unofficial].
## How it works
**Inputs:**
- `point`: Cartesian coordinates $(x, y, z)$ of the query location.
- `outVal`: Default output value if sampling fails.
- `triCubic`: Boolean flag selecting interpolation kernel (linear vs. cubic).
- `geomSize`: Dimensions of the bounding geometry $(G_x, G_y, G_z)$.
- `dataDim`: Dimensions of the volume texture $(D_x, D_y, D_z)$.
- `dataTex`: 3D texture object containing the volume data.

**Data Structures:**
- `float3`: A vector type with fields `x`, `y`, `z`.
- `cartToCyl(point)`: Helper function converting Cartesian $(x, y, z)$ to cylindrical $(r, \theta, z)$ [G].

**Steps:**
1. Calculate step sizes (voxel sizes in the cylindrical domain):
   $$ L_{step} = \frac{G_x}{D_x} $$
   $$ \Theta_{step} = \frac{G_y}{D_y} $$
   $$ Z_{step} = \frac{G_z}{D_z} $$
   [S1, unofficial]

2. Convert the Cartesian point to cylindrical coordinates:
   $$ \mathbf{c} = \text{cartToCyl}(\text{point}) $$
   Let $\mathbf{c} = (c_x, c_y, c_z)$, where $c_x$ is radius, $c_y$ is angle, and $c_z$ is height [G].

3. Normalize cylindrical coordinates to texture coordinates $\mathbf{t} = (t_x, t_y, t_z)$:
   $$ t_x = \frac{c_x}{L_{step}} $$
   $$ t_y = \frac{c_y}{\Theta_{step}} $$
   $$ t_z = \frac{c_z}{Z_{step}} $$
   [S1, unofficial]

4. Boundary Check:
   If any component of $\mathbf{t}$ is outside the valid range $[0, D_i - 1]$ for its respective dimension $i \in \{x, y, z\}$, return `outVal` immediately [S1, unofficial].

5. Sampling:
   - If `triCubic` is true, call `cubicInterpolate(t)` [S1, unofficial].
   - Otherwise, perform linear texture sampling:
     $$ \text{value} = \text{tex3D}(\text{dataTex}, t_x, t_y, t_z) $$
     [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def cart_to_cyl(point):
    x, y, z = point
    return (math.sqrt(x*x + y*y), math.atan2(y, x), z)

def sample_3d_linear(data_tex, tx, ty, tz):
    ix, iy, iz = int(tx), int(ty), int(tz)
    fx, fy, fz = tx - ix, ty - iy, tz - iz
    v000 = data_tex[ix, iy, iz]
    v100 = data_tex[ix+1, iy, iz]
    v010 = data_tex[ix, iy+1, iz]
    v110 = data_tex[ix+1, iy+1, iz]
    v001 = data_tex[ix, iy, iz+1]
    v101 = data_tex[ix+1, iy, iz+1]
    v011 = data_tex[ix, iy+1, iz+1]
    v111 = data_tex[ix+1, iy+1, iz+1]
    c00 = v000 + fx * (v100 - v000)
    c01 = v010 + fx * (v110 - v010)
    c10 = v001 + fx * (v101 - v001)
    c11 = v011 + fx * (v111 - v011)
    c0 = c00 + fy * (c01 - c00)
    c1 = c10 + fy * (c11 - c10)
    return c0 + fz * (c1 - c0)

def interpolate_cyl(point, out_val, tri_cubic, geom_size, data_dim, data_tex):
    length_step = geom_size.x / data_dim.width
    angle_step = geom_size.y / data_dim.height
    height_step = geom_size.z / data_dim.depth

    cyl_coord = cart_to_cyl(point)
    tex_coord = (cyl_coord[0] / length_step, cyl_coord[1] / angle_step, cyl_coord[2] / height_step)

    if (tex_coord[0] < 0 or tex_coord[0] > data_dim.width - 1 or
        tex_coord[1] < 0 or tex_coord[1] > data_dim.height - 1 or
        tex_coord[2] < 0 or tex_coord[2] > data_dim.depth - 1):
        return out_val

    if tri_cubic:
        # cubicInterpolate implementation not provided in sources [S1, unofficial]
        pass
    else:
        return sample_3d_linear(data_tex, tex_coord[0], tex_coord[1], tex_coord[2])
```
## Subtopics

### Spherical Coordinate Interpolation
Interpolation in spherical coordinates, mapping Cartesian points to spherical space before sampling [S1, unofficial]. It calculates step sizes for length, longitude, and latitude, converts the point, adds a Z-offset (`geomSize.z`) to the spherical coordinates, and maps them to texture coordinates: $X = \text{longitude}/\text{longStep}$, $Y = \text{latitude}/\text{latStep}$, $Z = \text{length}/\text{lengthStep}$ [S1, unofficial]. It uses the same linear or cubic sampling logic as the cylindrical variant and returns the output value if texture coordinates are out of bounds [S1, unofficial].

## Operating characteristics
**Time Complexity:** $O(1)$ per sample [G]. The conversion and normalization are constant-time arithmetic operations [G].
**Memory Complexity:** $O(1)$ auxiliary memory [G]. The method relies on the pre-allocated 3D texture `dataTex` [S1, unofficial].
**Strengths:** Aligns the sampling grid with cylindrical geometry [S1, unofficial].
**Limitations:** Requires cylindrical symmetry and assumes direct angle mapping without explicit discontinuity handling [S1, unofficial].
## Background
This method originates from a CUDA-based volume rendering implementation for the DTU dataset, specifically designed to handle different geometric types (cylindrical and spherical) within a unified framework [S1, unofficial]. It was part of an unofficial repository (`ojaisnielsen/dtu-2010-cuda-volume-rendering`) demonstrating GPU-accelerated ray casting with flexible coordinate transformations [S1, unofficial].
## Sources
[S1] ojaisnielsen/dtu-2010-cuda-volume-rendering, https://github.com/ojaisnielsen/dtu-2010-cuda-volume-rendering, fetched 2026-10-27 (unofficial)
