# Uniform Grid

Also called: Uniform Mesh

## Overview
A uniform grid is a structured mesh where vertices are spaced uniformly along each axis, with geometry and connectivity defined implicitly by bounding box limits and vertex counts [S1, unofficial].

## How it works
**Inputs and Outputs:**
- `aabb_min`: Minimum corner of the axis-aligned bounding box. Default: `[-1, -1, -1]` [S2, unofficial].
- `aabb_max`: Maximum corner of the axis-aligned bounding box. Default: `[1, 1, 1]` [S2, unofficial].
- `x_ext, y_ext, z_ext`: Number of vertices along each axis [S1, unofficial].
- `default_value`: Initial scalar value for all points. Default: `float('inf')` [S2, unofficial].
- `get_points()`: Returns a `(X, Y, Z, 3)` `float32` tensor containing 3D coordinates of all grid points [S2, unofficial].
- `get_values()`: Returns a `(X, Y, Z)` `float32` tensor containing scalar values at each vertex [S2, unofficial].
- `set_values(tensor)`: Updates the scalar field with a `(X, Y, Z)` `float32` tensor [S2, unofficial].

**Data Structure:**
The grid is defined by the positions of the minimum vertex $(x_0, y_0, z_0)$ and maximum vertex $(x_1, y_1, z_1)$, and the number of vertices along each axis $(x\_ext, y\_ext, z\_ext)$ [S1, unofficial]. Geometry and micro-topology are given implicitly; the connectivity of quadrilateral cells is computed trivially from the min/max positions and vertex counts [S1, unofficial]. Additional data attributes can be associated with each vertex [S1, unofficial].

**Algorithm:**
1. **Initialization**: Allocate memory for scalar values of size $x\_ext \times y\_ext \times z\_ext$ and fill with `default_value` [S2, unofficial].
2. **Coordinate Computation**: For any vertex at indices $(i, j, k)$ where $0 \le i < x\_ext$, $0 \le j < y\_ext$, $0 \le k < z\_ext$: [G]
   $$x = x_0 + i \cdot \frac{x_1 - x_0}{x\_ext - 1}$$ [G]
   $$y = y_0 + j \cdot \frac{y_1 - y_0}{y\_ext - 1}$$ [G]
   $$z = z_0 + k \cdot \frac{z_1 - z_0}{z\_ext - 1}$$ [G]
3. **Access**: Retrieve or update values via the `get_values()` and `set_values()` methods, which return or accept a `(X, Y, Z)` `float32` tensor [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class UniformGrid:
    def __init__(self, x_ext, y_ext, z_ext, aabb_min=(-1.0, -1.0, -1.0), aabb_max=(1.0, 1.0, 1.0), default_value=float('inf')):
        self.x_ext = x_ext
        self.y_ext = y_ext
        self.z_ext = z_ext
        self.aabb_min = aabb_min
        self.aabb_max = aabb_max
        self.values = [[[default_value for _ in range(z_ext)] for _ in range(y_ext)] for _ in range(x_ext)]

    def get_point(self, i, j, k):
        dx = self.aabb_max[0] - self.aabb_min[0] if self.x_ext > 1 else 0.0
        dy = self.aabb_max[1] - self.aabb_min[1] if self.y_ext > 1 else 0.0
        dz = self.aabb_max[2] - self.aabb_min[2] if self.z_ext > 1 else 0.0
        x = self.aabb_min[0] + i * (dx / (self.x_ext - 1)) if self.x_ext > 1 else self.aabb_min[0]
        y = self.aabb_min[1] + j * (dy / (self.y_ext - 1)) if self.y_ext > 1 else self.aabb_min[1]
        z = self.aabb_min[2] + k * (dz / (self.z_ext - 1)) if self.z_ext > 1 else self.aabb_min[2]
        return (x, y, z)

    def get_points(self):
        points = []
        for i in range(self.x_ext):
            for j in range(self.y_ext):
                for k in range(self.z_ext):
                    points.append(self.get_point(i, j, k))
        return points

    def get_values(self):
        return self.values

    def set_values(self, new_values):
        self.values = new_values
```

**Edge Cases and Pitfalls:**
- **Degenerate Dimensions**: If any extent is 1, the denominator $(extent - 1)$ becomes zero. The coordinate formula must handle this by returning the minimum bound for that axis [G].
- **Precision**: Floating-point accumulation errors may occur during coordinate computation for large extents; recomputing coordinates from indices per ray/cell is standard practice [G].

## Operating characteristics
- **Time Complexity**: Not given in the sources [G].
- **Memory Complexity**: Not given in the sources [G].
- **Strengths**: Trivial computation of coordinates and connectivity; suitable for regularly sampled data [S1, unofficial].
- **Limitations**: Cannot represent non-uniform sampling or complex geometries [S1, unofficial].
- **When to Use**: Regular, structured data like CT/MRI scans or terrain models [S1, unofficial].
- **When Not to Use**: Not given in the sources [G].

## Background
Uniform grids were introduced as a standard method for representing structured meshes in computational geometry and volume rendering [S1, unofficial]. They are widely used in digital terrain models and medical imaging (CT, MRI) due to their simplicity and efficiency [S1, unofficial].

## Sources
[S1] Multiresolution representation and compression of surfaces and volumes, https://doi.org/10.3929/ethz-a-004086690, fetched 2026-10-26 (unofficial)
[S2] GuangyanCai/isoext, https://github.com/GuangyanCai/isoext, fetched 2026-10-27 (unofficial)
