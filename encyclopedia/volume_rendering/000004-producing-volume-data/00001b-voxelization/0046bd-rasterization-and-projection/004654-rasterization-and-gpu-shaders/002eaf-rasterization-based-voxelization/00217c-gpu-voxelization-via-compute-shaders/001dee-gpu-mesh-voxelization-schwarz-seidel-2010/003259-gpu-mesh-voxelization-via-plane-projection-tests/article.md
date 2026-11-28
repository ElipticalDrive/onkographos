# GPU Mesh Voxelization via Plane/Projection Tests

Also called: GPU Voxelization via Plane and 2D Projection Tests, Solid Voxelization via Triangle Projection

## Overview
This method converts a triangle mesh into a solid 3D voxel grid on the GPU by explicitly testing every voxel within a triangle's bounding box against the triangle's plane equation and its 2D projections on the XY, YZ, and ZX planes [S2, unofficial]. It is distinguished from its sibling [GPU Voxelization via Projection and Bitmasking](13349-gpu-voxelization-via-projection-and-bitmasking.md) by using full 3D plane and projection tests for every voxel rather than a compact bitmask per voxel to store depth occupancy, making it suitable for general solid voxelization where explicit geometric containment is required [S1, unofficial][S2, unofficial].

## How it works
**Inputs:**
- `triangle_data`: A flat array of floats containing vertex positions. Each triangle consists of 9 floats ($v_{0x}, v_{0y}, v_{0z}, v_{1x}, \dots, v_{2z}$) [S2, unofficial].
- `voxel_table`: An array of `unsigned int` (32-bit) representing the voxel grid, where each bit corresponds to a voxel [S2, unofficial].
- `gridsize`: A 3D vector $(G_x, G_y, G_z)$ defining the resolution of the voxel grid [S2, unofficial].
- `bbox`: An axis-aligned bounding box (AABB) of the mesh in world coordinates, defined by `min` and `max` vectors [S2, unofficial].
- `morton_order`: A boolean flag determining if voxels are encoded using Morton codes (Z-order curve) or linear row-major indexing [S2, unofficial].
- `morton256_x`, `morton256_y`, `morton256_z`: Device constants (lookup tables) for Morton encoding, each 256 `uint32_t` elements, used if `morton_order` is true [S2, unofficial].

**Outputs:**
- `voxel_table`: Updated such that bits corresponding to filled voxels are set to 1 [S2, unofficial].

**Data Structures:**
- `AABox<T>`: A template struct with fields `min` and `max` of type $T$ (typically `float3`) [S2, unofficial].
- `voxinfo`: A struct containing `bbox` (AABox<float3>), `gridsize` (uint3), `n_triangles` (size_t), and `unit` (float3, the voxel size) [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class Vec3:
    def __init__(self, x, y, z): self.x, self.y, self.z = x, y, z
    def __add__(self, o): return Vec3(self.x+o.x, self.y+o.y, self.z+o.z)
    def __sub__(self, o): return Vec3(self.x-o.x, self.y-o.y, self.z-o.z)
    def __mul__(self, s): return Vec3(self.x*s, self.y*s, self.z*s)
    def __truediv__(self, s): return Vec3(self.x/s, self.y/s, self.z/s)
    def __neg__(self): return Vec3(-self.x, -self.y, -self.z)
    def dot(self, o): return self.x*o.x + self.y*o.y + self.z*o.z
    def cross(self, o): return Vec3(self.y*o.z - self.z*o.y, self.z*o.x - self.x*o.z, self.x*o.y - self.y*o.x)
    def length(self): return math.sqrt(self.dot(self))
    def normalize(self): l = self.length(); return self if l == 0 else self * (1.0/l)
    def min(self, o): return Vec3(min(self.x, o.x), min(self.y, o.y), min(self.z, o.z))
    def max(self, o): return Vec3(max(self.x, o.x), max(self.y, o.y), max(self.z, o.z))

class Vec2:
    def __init__(self, x, y): self.x, self.y = x, y
    def __add__(self, o): return Vec2(self.x+o.x, self.y+o.y)
    def __sub__(self, o): return Vec2(self.x-o.x, self.y-o.y)
    def __mul__(self, s): return Vec2(self.x*s, self.y*s)
    def dot(self, o): return self.x*o.x + self.y*o.y
    def __neg__(self): return Vec2(-self.x, -self.y)

def morton_encode(x, y, z, morton256_x, morton256_y, morton256_z):
    # Source uses 8-bit LUTs for Morton encoding [S2, unofficial]
    return (morton256_x[x & 0xFF] | 
            (morton256_y[y & 0xFF] << 8) | 
            (morton256_z[z & 0xFF] << 16))

def check_point_triangle(p, v0, v1, v2):
    # Standard barycentric coordinate test [G]
    d1 = (p.x - v2.x) * (v0.y - v2.y) - (p.y - v2.y) * (v0.x - v2.x)
    d2 = (p.x - v0.x) * (v1.y - v0.y) - (p.y - v0.y) * (v1.x - v0.x)
    d3 = (p.x - v1.x) * (v2.y - v1.y) - (p.y - v1.y) * (v2.x - v1.x)
    has_neg = (d1 < 0) or (d2 < 0) or (d3 < 0)
    has_pos = (d1 > 0) or (d2 > 0) or (d3 > 0)
    return 0 if (has_neg and has_pos) else 1

def set_bit(voxel_table, index, morton_order, morton256_x, morton256_y, morton256_z, gridsize):
    if morton_order:
        location = morton_encode(index.x, index.y, index.z, morton256_x, morton256_y, morton256_z)
    else:
        location = index.x + index.y * gridsize.x + index.z * (gridsize.y * gridsize.x)
    int_location = location // 32
    bit_pos = 31 - (location % 32)
    voxel_table[int_location] |= (1 << bit_pos)

def voxelize_kernel(triangle_data, voxel_table, info, morton256_x, morton256_y, morton256_z):
    thread_id = 0 # Simplified for Python; in CUDA: threadIdx.x + blockIdx.x * blockDim.x
    stride = 1 # Simplified
    while thread_id < info.n_triangles:
        t = thread_id * 9
        v0 = Vec3(triangle_data[t], triangle_data[t+1], triangle_data[t+2]) - info.bbox.min
        v1 = Vec3(triangle_data[t+3], triangle_data[t+4], triangle_data[t+5]) - info.bbox.min
        v2 = Vec3(triangle_data[t+6], triangle_data[t+7], triangle_data[t+8]) - info.bbox.min
        
        e0 = v1 - v0
        e1 = v2 - v1
        e2 = v0 - v2
        
        n = (e0.cross(e1)).normalize()
        
        if abs(n.x) < 1e-6:
            thread_id += stride
            continue
            
        c = Vec3(info.unit.x if n.x > 0.0 else 0.0, info.unit.y if n.y > 0.0 else 0.0, info.unit.z if n.z > 0.0 else 0.0)
        
        d1 = n.dot(c - v0)
        delta_p = info.unit
        d2 = n.dot((delta_p - c) - v0)
        
        n_xy_e0 = Vec2(-1.0 * e0.y, e0.x)
        n_xy_e1 = Vec2(-1.0 * e1.y, e1.x)
        n_xy_e2 = Vec2(-1.0 * e2.y, e2.x)
        if n.z < 0.0:
            n_xy_e0 = -n_xy_e0
            n_xy_e1 = -n_xy_e1
            n_xy_e2 = -n_xy_e2
            
        d_xy_e0 = (-1.0 * n_xy_e0.dot(Vec2(v0.x, v0.y))) + max(0.0, info.unit.x * n_xy_e0.x) + max(0.0, info.unit.y * n_xy_e0.y)
        d_xy_e1 = (-1.0 * n_xy_e1.dot(Vec2(v1.x, v1.y))) + max(0.0, info.unit.x * n_xy_e1.x) + max(0.0, info.unit.y * n_xy_e1.y)
        d_xy_e2 = (-1.0 * n_xy_e2.dot(Vec2(v2.x, v2.y))) + max(0.0, info.unit.x * n_xy_e2.x) + max(0.0, info.unit.y * n_xy_e2.y)
        
        n_yz_e0 = Vec2(-1.0 * e0.z, e0.y)
        n_yz_e1 = Vec2(-1.0 * e1.z, e1.y)
        n_yz_e2 = Vec2(-1.0 * e2.z, e2.y)
        if n.x < 0.0:
            n_yz_e0 = -n_yz_e0
            n_yz_e1 = -n_yz_e1
            n_yz_e2 = -n_yz_e2
            
        d_yz_e0 = (-1.0 * n_yz_e0.dot(Vec2(v0.y, v0.z))) + max(0.0, info.unit.y * n_yz_e0.y) + max(0.0, info.unit.z * n_yz_e0.z)
        d_yz_e1 = (-1.0 * n_yz_e1.dot(Vec2(v1.y, v1.z))) + max(0.0, info.unit.y * n_yz_e1.y) + max(0.0, info.unit.z * n_yz_e1.z)
        d_yz_e2 = (-1.0 * n_yz_e2.dot(Vec2(v2.y, v2.z))) + max(0.0, info.unit.y * n_yz_e2.y) + max(0.0, info.unit.z * n_yz_e2.z)
        
        n_zx_e0 = Vec2(-1.0 * e0.x, e0.z)
        n_zx_e1 = Vec2(-1.0 * e1.x, e1.z)
        n_zx_e2 = Vec2(-1.0 * e2.x, e2.z)
        if n.y < 0.0:
            n_zx_e0 = -n_zx_e0
            n_zx_e1 = -n_zx_e1
            n_zx_e2 = -n_zx_e2
            
        d_zx_e0 = (-1.0 * n_zx_e0.dot(Vec2(v0.z, v0.x))) + max(0.0, info.unit.z * n_zx_e0.z) + max(0.0, info.unit.x * n_zx_e0.x)
        d_zx_e1 = (-1.0 * n_zx_e1.dot(Vec2(v1.z, v1.x))) + max(0.0, info.unit.z * n_zx_e1.z) + max(0.0, info.unit.x * n_zx_e1.x)
        d_zx_e2 = (-1.0 * n_zx_e2.dot(Vec2(v2.z, v2.x))) + max(0.0, info.unit.z * n_zx_e2.z) + max(0.0, info.unit.x * n_zx_e2.x)
        
        t_bbox_world_min = v0.min(v1.min(v2))
        t_bbox_world_max = v0.max(v1.max(v2))
        
        t_bbox_grid_min = t_bbox_world_min / info.unit
        t_bbox_grid_max = t_bbox_world_max / info.unit
        t_bbox_grid_min = Vec3(max(0, int(t_bbox_grid_min.x)), max(0, int(t_bbox_grid_min.y)), max(0, int(t_bbox_grid_min.z)))
        t_bbox_grid_max = Vec3(min(info.gridsize.x, int(t_bbox_grid_max.x)), min(info.gridsize.y, int(t_bbox_grid_max.y)), min(info.gridsize.z, int(t_bbox_grid_max.z)))
        
        for z in range(t_bbox_grid_min.z, t_bbox_grid_max.z + 1):
            for y in range(t_bbox_grid_min.y, t_bbox_grid_max.y + 1):
                for x in range(t_bbox_grid_min.x, t_bbox_grid_max.x + 1):
                    p = Vec3(x * info.unit.x, y * info.unit.y, z * info.unit.z)
                    nDOTp = n.dot(p)
                    
                    if (nDOTp + d1) * (nDOTp + d2) > 0.0:
                        continue
                        
                    p_xy = Vec2(p.x, p.y)
                    if (n_xy_e0.dot(p_xy) + d_xy_e0 < 0.0) or \
                       (n_xy_e1.dot(p_xy) + d_xy_e1 < 0.0) or \
                       (n_xy_e2.dot(p_xy) + d_xy_e2 < 0.0):
                        continue
                        
                    p_yz = Vec2(p.y, p.z)
                    if (n_yz_e0.dot(p_yz) + d_yz_e0 < 0.0) or \
                       (n_yz_e1.dot(p_yz) + d_yz_e1 < 0.0) or \
                       (n_yz_e2.dot(p_yz) + d_yz_e2 < 0.0):
                        continue
                        
                    p_zx = Vec2(p.z, p.x)
                    if (n_zx_e0.dot(p_zx) + d_zx_e0 < 0.0) or \
                       (n_zx_e1.dot(p_zx) + d_zx_e1 < 0.0) or \
                       (n_zx_e2.dot(p_zx) + d_zx_e2 < 0.0):
                        continue
                        
                    set_bit(voxel_table, Vec3(x, y, z), info.morton_order, morton256_x, morton256_y, morton256_z, info.gridsize)
                    
        thread_id += stride
```

**Alternative Projection-Based Variant (from S2 notes):**
The sources also describe a variant that projects triangles onto the YZ plane, iterates over the 2D bounding box, and fills voxels along the X-axis up to the intersection [S2, unofficial].
1. Project vertices to YZ plane: $v_{yz} = (v.y, v.z)$ [S2, unofficial].
2. Ensure counter-clockwise winding in YZ plane [S2, unofficial].
3. Compute 2D bounding box in grid indices [S2, unofficial].
4. Iterate over $(y, z)$ in the 2D box [S2, unofficial].
5. Check if voxel center $(y, z)$ is inside the projected triangle [S2, unofficial].
6. Calculate max X index ($x_{max}$) by intersecting a ray from the voxel center along the X-axis with the triangle plane: $x_{max} = \lfloor \frac{\text{get\_x\_coordinate}(n, v_0, \text{point})}{\text{info.unit.x}} - 0.5 \rfloor$ [S2, unofficial].
7. Iterate $x$ from 0 to $x_{max}$ and set bits [S2, unofficial].

## Subtopics

### [GPU Voxelization via Projection and Bitmasking](13349-gpu-voxelization-via-projection-and-bitmasking.md)
Converts a mesh to a voxel grid by rasterizing triangles into a bitmask representation. It projects triangles onto 2D planes and tests voxel occupancy against triangle edges using plane equations, updating a 32-bit integer bitmask per voxel to represent occupancy along the depth axis (Z). It is distinguished by using a compact bitmask per voxel to store depth occupancy, reducing memory usage compared to storing a boolean or float per voxel, which is efficient for sparse or thin geometry.

## Operating characteristics
Not given in the sources.

## Background
It builds on [GPU Mesh Voxelization (Schwarz & Seidel 2010)](1633-gpu-mesh-voxelization-schwarz-seidel-2010.md) by using compute shaders or CUDA kernels to parallelize the voxel testing process [S2, unofficial]. The specific plane and projection test approach is detailed in unofficial sources related to CUDA voxelization implementations [S2, unofficial].

## Sources
[S1] GilesPenfold/MasterThesisSVO, https://github.com/GilesPenfold/MasterThesisSVO, fetched 2026-10-26 (unofficial)
[S2] GitHub - Forceflow/cuda_voxelizer: CUDA Voxelizer to convert polygon ..., https://github.com/Forceflow/cuda_voxelizer, fetched 2026-10-27 (unofficial)
