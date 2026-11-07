# Voxel Interpolation (Nearest, Trilinear, Triquadratic, Tricubic)
Also called: Nearest Neighbor Sampler, SampleFromVoxels, Tricubic Sampler, Trilinear Sampler, Triquadratic Sampler

## Overview
This method provides a unified interface for sampling values from a dense or sparse voxel grid at arbitrary floating-point coordinates using interpolation orders 0 (nearest), 1 (trilinear), 2 (triquadratic), and 3 (tricubic) [S1, unofficial]. It exposes a templated API that abstracts the underlying tree structure (e.g., OpenVDB/NanoVDB) while supporting multiple precision levels and interpolation kernels, whereas [Voxel Trilinear Interpolation (SIMD)](2564-voxel-trilinear-interpolation-simd.md) is hardcoded to trilinear with AVX optimizations [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A voxel grid tree (`TreeT`) and a sampling coordinate (`Vec3f` or `Vec3d`) [S1, unofficial].
- Output: The interpolated scalar value at the coordinate [G].

**Data Structures:**
The sampler uses a read accessor (`cnanovdb_readaccessor`) to cache node lookups during tree traversal [S1, unofficial]. It relies on the underlying tree's node layout (e.g., child masks, value masks, and offsets) for efficient access [S1, unofficial].

**Parameters:**
- `Order`: Integer specifying interpolation degree. 0=Nearest, 1=Trilinear, 2=Triquadratic, 3=Tricubic [S1, unofficial].
- `TreeT`: The specific tree type (e.g., NanoVDB tree) [S1, unofficial].

**Steps:**
1. **Initialization:** Create a sampler instance using the factory function `math::createSampler<Order, TreeT, false>(grid.tree())` [S1, unofficial].
2. **Coordinate Handling:** The sampler accepts integer coordinates (`CoordT ijk`) or floating-point coordinates (`Vec3f xyz` / `Vec3d xyz`) [S1, unofficial].
3. **Lookup:** For floating-point coordinates, compute the base integer indices and traverse the tree using the accessor to locate the active node [S1, unofficial].
4. **Interpolation:** Apply the polynomial kernel corresponding to the specified order [G].
5. **Return:** Return the computed value; return 0 if the coordinate is outside the volume bounds [G].

**Formulas:**
Let $f_x, f_y, f_z$ be the fractional distances from the base integer coordinate $(i, j, k)$, where $0 \le f < 1$.
- **Order 0 (Nearest):** $V = V_{i,j,k}$ [G].
- **Order 1 (Trilinear):** $V = \sum_{d_x,d_y,d_z \in \{0,1\}} w_x(d_x) w_y(d_y) w_z(d_z) V_{i+d_x, j+d_y, k+d_z}$, where $w_x(0) = 1-f_x$ and $w_x(1) = f_x$ [G].
- **Order 2 (Triquadratic):** Uses a $3 \times 3 \times 3$ grid of voxels with quadratic basis functions $w_x(d) = \frac{1}{2}d(d-1)$ for $d \in \{-1, 0, 1\}$, and $w_x(0) = 1 - \frac{3}{2}f_x^2 + f_x^3$ [G].
- **Order 3 (Tricubic):** Uses a $4 \times 4 \times 4$ grid of voxels with cubic convolution kernel weights $w_x(d) = \frac{2}{3}|d|^3 - |d|^2 + \frac{1}{2}$ for $|d| \le 1$, and $w_x(d) = \frac{1}{6}(-|d|^3 + 5|d|^2 - 8|d| + 4)$ for $1 < |d| \le 2$ [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class Vec3:
    def __init__(self, x, y, z):
        self.x, self.y, self.z = x, y, z
    def __getitem__(self, i): return (self.x, self.y, self.z)[i]
    def __add__(self, other): return Vec3(self.x + other.x, self.y + other.y, self.z + other.z)
    def __sub__(self, other): return Vec3(self.x - other.x, self.y - other.y, self.z - other.z)

class Coord:
    def __init__(self, i, j, k):
        self.i, self.j, self.k = i, j, k
    def __getitem__(self, i): return (self.i, self.j, self.k)[i]
    def __add__(self, other): return Coord(self.i + other.i, self.j + other.j, self.k + other.k)

def lerp(a, b, t):
    return a + t * (b - a)

def sample_voxel(tree, coord):
    if coord.i < 0 or coord.j < 0 or coord.k < 0:
        return 0.0
    return tree.get_value(coord)

class VoxelSampler:
    def __init__(self, grid_tree, order):
        self.tree = grid_tree
        self.order = order
        self.accessor = []  # Internal cache for node lookups [S1, unofficial]

    def sample_float(self, xyz: Vec3) -> float:
        ijk_base = Coord(int(math.floor(xyz.x)), int(math.floor(xyz.y)), int(math.floor(xyz.z)))
        fx, fy, fz = xyz.x - ijk_base.i, xyz.y - ijk_base.j, xyz.z - ijk_base.k
        
        node = self.tree.get_node(ijk_base)  # Traverses tree using accessor [S1, unofficial]
        if node is None:
            return 0.0  # Boundary check [G]
            
        if self.order == 0:
            return sample_voxel(self.tree, ijk_base)
            
        elif self.order == 1:
            v000 = sample_voxel(self.tree, ijk_base)
            v100 = sample_voxel(self.tree, ijk_base + Coord(1,0,0))
            v010 = sample_voxel(self.tree, ijk_base + Coord(0,1,0))
            v001 = sample_voxel(self.tree, ijk_base + Coord(0,0,1))
            v110 = sample_voxel(self.tree, ijk_base + Coord(1,1,0))
            v101 = sample_voxel(self.tree, ijk_base + Coord(1,0,1))
            v011 = sample_voxel(self.tree, ijk_base + Coord(0,1,1))
            v111 = sample_voxel(self.tree, ijk_base + Coord(1,1,1))
            ix00 = lerp(v000, v100, fx)
            ix01 = lerp(v001, v101, fx)
            ix10 = lerp(v010, v110, fx)
            ix11 = lerp(v011, v111, fx)
            iy0 = lerp(ix00, ix01, fz)
            iy1 = lerp(ix10, ix11, fz)
            return lerp(iy0, iy1, fy)
            
        elif self.order == 2:
            # Triquadratic: 3x3x3 grid, indices from -1 to 1
            vals = []
            for dx in (-1, 0, 1):
                for dy in (-1, 0, 1):
                    for dz in (-1, 0, 1):
                        vals.append(sample_voxel(self.tree, ijk_base + Coord(dx, dy, dz)))
            # Quadratic weights
            def w2(d, f):
                if d == 0: return 1 - 1.5*f*f + f*f*f
                if d == 1: return 0.5*f*f*(3 - 2*f)
                return 0.5*f*f*(1 + f)
            res = 0.0
            idx = 0
            for dx in (-1, 0, 1):
                for dy in (-1, 0, 1):
                    for dz in (-1, 0, 1):
                        res += vals[idx] * w2(dx, fx) * w2(dy, fy) * w2(dz, fz)
                        idx += 1
            return res
            
        elif self.order == 3:
            # Tricubic: 4x4x4 grid, indices from 0 to 3
            vals = []
            for dx in (0, 1, 2, 3):
                for dy in (0, 1, 2, 3):
                    for dz in (0, 1, 2, 3):
                        vals.append(sample_voxel(self.tree, ijk_base + Coord(dx, dy, dz)))
            def w3(d, f):
                abs_d = abs(d)
                if abs_d <= 1: return (2.0/3.0)*abs_d**3 - abs_d**2 + 0.5
                if abs_d <= 2: return (1.0/6.0)*(-abs_d**3 + 5*abs_d**2 - 8*abs_d + 4)
                return 0.0
            res = 0.0
            idx = 0
            for dx in (0, 1, 2, 3):
                for dy in (0, 1, 2, 3):
                    for dz in (0, 1, 2, 3):
                        res += vals[idx] * w3(dx-1, fx) * w3(dy-1, fy) * w3(dz-1, fz)
                        idx += 1
            return res
            
        else:
            raise ValueError("Unsupported order")

    def sample_int(self, ijk: Coord) -> float:
        return sample_voxel(self.tree, ijk)
```

## Subtopics

### Nearest Neighbor (Order 0)
Returns the value of the voxel at the nearest integer coordinate [G]. Fastest execution, but produces blocky artifacts [G]. Use for low-resolution data or binary masks [G].

### Trilinear (Order 1)
Computes the weighted average of the 8 surrounding voxels based on fractional distances [G]. Standard balance of speed and smoothness for general volume rendering [G].

### Triquadratic (Order 2)
Fits a quadratic surface using a $3 \times 3 \times 3$ grid of voxels [G]. Provides higher continuity ($C^1$) than linear interpolation [G]. Use when smooth gradients are critical [G].

### Tricubic (Order 3)
Fits a cubic surface using a $4 \times 4 \times 4$ grid of voxels [G]. Provides the highest continuity ($C^2$) but requires 64 lookups per sample [G]. Use for high-fidelity scientific visualization [G].

## Operating characteristics
- **Time Complexity:** O(1) for tree lookup assuming fixed depth, but interpolation cost scales with order: 1 sample for nearest, 8 for trilinear, 27 for triquadratic, and 64 for tricubic [G].
- **Memory:** Depends on the underlying tree structure; the sampler maintains a small read-accessor cache for node lookups [S1, unofficial].
- **Strengths:** Unified templated API supporting multiple interpolation orders and both integer and floating-point coordinates [S1, unofficial].
- **Limitations:** Higher orders increase memory bandwidth and computation; boundary conditions for higher-order interpolations require explicit clamping or extrapolation [G].

## Background
The method is provided by the NanoVDB/HNanoSolver codebase as a C++ templated class for sampling voxel grid values [S1, unofficial]. It is designed to work with OpenVDB/NanoVDB tree structures, leveraging their internal node layout for efficient traversal [S1, unofficial].

## Sources
[S1] ZephirFXEC/HNanoSolver, https://github.com/ZephirFXEC/HNanoSolver, fetched 2026-10-27 (unofficial)
[S2] OpenVDB: nanovdb::math Namespace Reference, https://openvdb.org/documentation/doxygen/namespacenanovdb_1_1math.html, fetched 2026-10-24
