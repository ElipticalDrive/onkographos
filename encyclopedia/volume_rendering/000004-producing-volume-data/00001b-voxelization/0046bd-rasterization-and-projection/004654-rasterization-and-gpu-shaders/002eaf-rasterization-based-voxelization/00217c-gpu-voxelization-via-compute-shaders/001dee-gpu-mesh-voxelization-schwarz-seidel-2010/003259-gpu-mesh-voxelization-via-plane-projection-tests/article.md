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
# Helper: Morton encoding using lookup tables
def morton_encode(x, y, z, morton256_x, morton256_y, morton256_z):
    # Assumes x, y, z are within 0-255 range for single byte lookup
    # In practice, this is extended for larger grids or used recursively
    # Here we assume the source uses a specific LUT-based implementation
    # for indices < 256. For general implementation, standard bit interleaving is used.
    # The source provides LUTs for 8-bit chunks.
    # Simplified for the scope of the source:
    return (morton256_x[x & 0xFF] | 
            (morton256_y[y & 0xFF] << 8) | 
            (morton256_z[z & 0xFF] << 16))

# Helper: Check if point is inside 2D triangle
def check_point_triangle(p, v0, v1, v2):
    # Returns 1 if inside, 2 if on edge, 3 if vertex, 0 if outside
    # Implementation details omitted as per source "check_point_triangle"
    # Standard barycentric or cross-product method
    pass

# Helper: Set bit in voxel table
def set_bit(voxel_table, index, morton_order, morton256_x, morton256_y, morton256_z, gridsize):
    if morton_order:
        location = morton_encode(index.x, index.y, index.z, morton256_x, morton256_y, morton256_z)
    else:
        location = index.x + index.y * gridsize.x + index.z * (gridsize.y * gridsize.x)
    
    int_location = location // 32
    bit_pos = 31 - (location % 32)
    voxel_table[int_location] |= (1 << bit_pos)

# Main Kernel Logic
def voxelize_kernel(triangle_data, voxel_table, info, morton256_x, morton256_y, morton256_z):
    thread_id = threadIdx.x + blockIdx.x * blockDim.x
    stride = blockDim.x * gridDim.x
    
    while thread_id < info.n_triangles:
        t = thread_id * 9
        
        # 1. Transform vertices to local coordinates relative to bounding box min
        v0 = float3(triangle_data[t], triangle_data[t+1], triangle_data[t+2]) - info.bbox.min
        v1 = float3(triangle_data[t+3], triangle_data[t+4], triangle_data[t+5]) - info.bbox.min
        v2 = float3(triangle_data[t+6], triangle_data[t+7], triangle_data[t+8]) - info.bbox.min
        
        # 2. Compute edge vectors
        e0 = v1 - v0
        e1 = v2 - v1
        e2 = v0 - v2
        
        # 3. Compute triangle normal
        n = normalize(cross(e0, e1))
        
        # 4. Pitfall: Skip if parallel to YZ plane (normal X component near zero)
        if abs(n.x) < float_error:
            thread_id += stride
            continue
            
        # 5. Compute plane test offset vector c
        c = float3(
            info.unit.x if n.x > 0.0f else 0.0f,
            info.unit.y if n.y > 0.0f else 0.0f,
            info.unit.z if n.z > 0.0f else 0.0f
        )
        
        # 6. Compute plane test distances
        # d1: distance from v0 to plane, adjusted for voxel size
        d1 = dot(n, (c - v0))
        # d2: distance from (v0 + unit) to plane, adjusted
        delta_p = info.unit
        d2 = dot(n, ((delta_p - c) - v0))
        
        # 7. Prepare XY projection test
        n_xy_e0 = float2(-1.0f * e0.y, e0.x)
        n_xy_e1 = float2(-1.0f * e1.y, e1.x)
        n_xy_e2 = float2(-1.0f * e2.y, e2.x)
        if n.z < 0.0f:
            n_xy_e0 = -n_xy_e0
            n_xy_e1 = -n_xy_e1
            n_xy_e2 = -n_xy_e2
            
        d_xy_e0 = (-1.0f * dot(n_xy_e0, float2(v0.x, v0.y))) + max(0.0f, info.unit.x * n_xy_e0.x) + max(0.0f, info.unit.y * n_xy_e0.y)
        d_xy_e1 = (-1.0f * dot(n_xy_e1, float2(v1.x, v1.y))) + max(0.0f, info.unit.x * n_xy_e1.x) + max(0.0f, info.unit.y * n_xy_e1.y)
        d_xy_e2 = (-1.0f * dot(n_xy_e2, float2(v2.x, v2.y))) + max(0.0f, info.unit.x * n_xy_e2.x) + max(0.0f, info.unit.y * n_xy_e2.y)
        
        # 8. Prepare YZ projection test
        n_yz_e0 = float2(-1.0f * e0.z, e0.y)
        n_yz_e1 = float2(-1.0f * e1.z, e1.y)
        n_yz_e2 = float2(-1.0f * e2.z, e2.y)
        if n.x < 0.0f:
            n_yz_e0 = -n_yz_e0
            n_yz_e1 = -n_yz_e1
            n_yz_e2 = -n_yz_e2
            
        d_yz_e0 = (-1.0f * dot(n_yz_e0, float2(v0.y, v0.z))) + max(0.0f, info.unit.y * n_yz_e0.y) + max(0.0f, info.unit.z * n_yz_e0.z)
        d_yz_e1 = (-1.0f * dot(n_yz_e1, float2(v1.y, v1.z))) + max(0.0f, info.unit.y * n_yz_e1.y) + max(0.0f, info.unit.z * n_yz_e1.z)
        d_yz_e2 = (-1.0f * dot(n_yz_e2, float2(v2.y, v2.z))) + max(0.0f, info.unit.y * n_yz_e2.y) + max(0.0f, info.unit.z * n_yz_e2.z)
        
        # 9. Prepare ZX projection test
        n_zx_e0 = float2(-1.0f * e0.x, e0.z)
        n_zx_e1 = float2(-1.0f * e1.x, e1.z)
        n_zx_e2 = float2(-1.0f * e2.x, e2.z)
        if n.y < 0.0f:
            n_zx_e0 = -n_zx_e0
            n_zx_e1 = -n_zx_e1
            n_zx_e2 = -n_zx_e2
            
        d_zx_e0 = (-1.0f * dot(n_zx_e0, float2(v0.z, v0.x))) + max(0.0f, info.unit.z * n_zx_e0.z) + max(0.0f, info.unit.x * n_zx_e0.x)
        d_zx_e1 = (-1.0f * dot(n_zx_e1, float2(v1.z, v1.x))) + max(0.0f, info.unit.z * n_zx_e1.z) + max(0.0f, info.unit.x * n_zx_e1.x)
        d_zx_e2 = (-1.0f * dot(n_zx_e2, float2(v2.z, v2.x))) + max(0.0f, info.unit.z * n_zx_e2.z) + max(0.0f, info.unit.x * n_zx_e2.x)
        
        # 10. Compute triangle bounding box in grid coordinates
        t_bbox_world = AABox(float3(min(v0.x, min(v1.x, v2.x)), min(v0.y, min(v1.y, v2.y)), min(v0.z, min(v1.z, v2.z))),
                             float3(max(v0.x, max(v1.x, v2.x)), max(v0.y, max(v1.y, v2.y)), max(v0.z, max(v1.z, v2.z))))
        
        t_bbox_grid_min = clamp(float3_to_int3(t_bbox_world.min / info.unit), int3(0, 0, 0), info.gridsize)
        t_bbox_grid_max = clamp(float3_to_int3(t_bbox_world.max / info.unit), int3(0, 0, 0), info.gridsize)
        
        # 11. Loop over voxels in bounding box
        for z in range(t_bbox_grid_min.z, t_bbox_grid_max.z + 1):
            for y in range(t_bbox_grid_min.y, t_bbox_grid_max.y + 1):
                for x in range(t_bbox_grid_min.x, t_bbox_grid_max.x + 1):
                    
                    # Compute voxel center in local coordinates
                    p = float3(x * info.unit.x, y * info.unit.y, z * info.unit.z)
                    nDOTp = dot(n, p)
                    
                    # Plane Test: Check if voxel is outside the slab defined by the triangle thickness
                    if (nDOTp + d1) * (nDOTp + d2) > 0.0f:
                        continue
                        
                    # XY Projection Test
                    p_xy = float2(p.x, p.y)
                    if (dot(n_xy_e0, p_xy) + d_xy_e0 < 0.0f) or \
                       (dot(n_xy_e1, p_xy) + d_xy_e1 < 0.0f) or \
                       (dot(n_xy_e2, p_xy) + d_xy_e2 < 0.0f):
                        continue
                        
                    # YZ Projection Test
                    p_yz = float2(p.y, p.z)
                    if (dot(n_yz_e0, p_yz) + d_yz_e0 < 0.0f) or \
                       (dot(n_yz_e1, p_yz) + d_yz_e1 < 0.0f) or \
                       (dot(n_yz_e2, p_yz) + d_yz_e2 < 0.0f):
                        continue
                        
                    # ZX Projection Test
                    p_zx = float2(p.z, p.x)
                    if (dot(n_zx_e0, p_zx) + d_zx_e0 < 0.0f) or \
                       (dot(n_zx_e1, p_zx) + d_zx_e1 < 0.0f) or \
                       (dot(n_zx_e2, p_zx) + d_zx_e2 < 0.0f):
                        continue
                        
                    # All tests passed, set the voxel
                    set_bit(voxel_table, int3(x, y, z), info.morton_order, morton256_x, morton256_y, morton256_z, info.gridsize)
                    
        thread_id += stride
```

**Alternative Projection-Based Variant (from S2 notes):**
The sources also describe a variant that projects triangles onto the YZ plane, iterates over the 2D bounding box, and fills voxels along the X-axis up to the intersection.
1. Project vertices to YZ plane: $v_{yz} = (v.y, v.z)$.
2. Ensure counter-clockwise winding in YZ plane.
3. Compute 2D bounding box in grid indices.
4. Iterate over $(y, z)$ in the 2D box.
5. Check if voxel center $(y, z)$ is inside the projected triangle.
6. Calculate max X index ($x_{max}$) by intersecting a ray from the voxel center along the X-axis with the triangle plane: $x_{max} = \lfloor \frac{\text{get\_x\_coordinate}(n, v_0, \text{point})}{\text{info.unit.x}} - 0.5 \rfloor$.
7. Iterate $x$ from 0 to $x_{max}$ and set bits.

## Subtopics

### [GPU Voxelization via Projection and Bitmasking](13349-gpu-voxelization-via-projection-and-bitmasking.md)
Converts a mesh to a voxel grid by rasterizing triangles into a bitmask representation. It projects triangles onto 2D planes and tests voxel occupancy against triangle edges using plane equations, updating a 32-bit integer bitmask per voxel to represent occupancy along the depth axis (Z). It is distinguished by using a compact bitmask per voxel to store depth occupancy, reducing memory usage compared to storing a boolean or float per voxel, which is efficient for sparse or thin geometry.

## Operating characteristics
- **Time Complexity:** $O(N \cdot V_{bbox})$, where $N$ is the number of triangles and $V_{bbox}$ is the average number of voxels in a triangle's bounding box. Each voxel requires multiple dot products and comparisons.
- **Memory Complexity:** $O(G_x \cdot G_y \cdot G_z / 32)$ for the bit-packed voxel table, plus $O(N)$ for triangle data and $O(1)$ for per-thread constants [S2, unofficial].
- **Performance:** Performance depends heavily on the sparsity of the mesh and the size of the bounding boxes. Large triangles covering many voxels will be slow due to the inner loop. The use of atomic OR or bit-setting operations can introduce contention if multiple threads write to the same voxel word, though the bit-packing reduces the total number of writes compared to byte/float arrays.
- **Strengths:** Explicit geometric tests ensure accurate solid voxelization. The bit-packing reduces memory footprint.
- **Limitations:** Inefficient for very large bounding boxes with low triangle coverage. The projection variant requires careful handling of the X-axis intersection calculation.
- **When to use:** When solid, watertight voxelization is required and memory is a constraint.
- **When not to use:** When the mesh is extremely sparse and the bounding boxes are large relative to the grid, or when a faster, approximate rasterization-based method is sufficient.

## Background
This method is part of the GPU-accelerated voxelization techniques developed to convert triangle meshes into voxel grids for volume rendering and other applications. It builds on [GPU Mesh Voxelization (Schwarz & Seidel 2010)](1633-gpu-mesh-voxelization-schwarz-seidel-2010.md) by using compute shaders or CUDA kernels to parallelize the voxel testing process. The specific plane and projection test approach is detailed in unofficial sources related to CUDA voxelization implementations [S2, unofficial].

## Sources
[S1] GilesPenfold/MasterThesisSVO, https://github.com/GilesPenfold/MasterThesisSVO, fetched 2026-10-26 (unofficial)
[S2] GitHub - Forceflow/cuda_voxelizer: CUDA Voxelizer to convert polygon ..., https://github.com/Forceflow/cuda_voxelizer, fetched 2026-10-27 (unofficial)
