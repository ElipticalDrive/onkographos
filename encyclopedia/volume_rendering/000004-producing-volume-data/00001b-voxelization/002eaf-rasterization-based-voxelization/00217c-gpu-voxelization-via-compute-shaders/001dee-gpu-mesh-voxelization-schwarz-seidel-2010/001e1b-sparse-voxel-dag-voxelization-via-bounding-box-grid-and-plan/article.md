# Sparse Voxel DAG Voxelization via Bounding Box Grid and Plane Projections

Also called: Morton-coded Voxel Filling, Triangle Voxelization with XY/ZX/YZ Projection Tests

## Overview
This method voxelizes a triangle mesh into a Sparse Voxel DAG by rasterizing triangles into a Morton-coded voxel subgrid [S1, unofficial]. It iterates over a 2D grid (XY) within a triangle's bounding box, determines the Z-range of voxels intersecting the triangle plane, and fills only those voxels that pass containment tests in the XY, ZX, and YZ projections [S1, unofficial].

## How it works
**Inputs:**
- A triangle $t$ with vertices $v_0, v_1, v_2$ and associated colors $c_0, c_1, c_2$ [S1, unofficial].
- A global bounding box grid $p\_bbox\_grid$ defining the voxelization domain $[min[Z], max[Z]]$ in Z [S1, unofficial].
- A partition bounding box grid $t\_bbox\_grid$ defining the current subgrid range $[min[X], max[X]]$ and $[min[Y], max[Y]]$ [S1, unofficial].
- `unitlength`: The size of a single voxel in world units [S1, unofficial].
- `morton_start`: An offset added to the Morton index to map it to the correct position in the voxels array [S1, unofficial].
- `voxels`: A sparse array or bitfield where `EMPTY_VOXEL` indicates an unoccupied cell [S1, unofficial].
- `voxel_data`: A list to store non-binary voxel information (optional, for binary voxelization this step is skipped) [S1, unofficial].

**Outputs:**
- Updated `voxels` array with filled entries [S1, unofficial].
- Updated `voxel_data` list (if non-binary) [S1, unofficial].
- `nfilled`: Count of newly filled voxels [S1, unofficial].

**Data Structures:**
- `VoxelData`: Struct containing `morton` (`uint_fast64_t`), `color` (`vec3`), and `normal` (`vec3`) [S1, unofficial]. Total size is `sizeof(uint_fast64_t) + 2 * (3 * sizeof(float))` [S1, unofficial].
- `AABox`: Axis-aligned bounding box defined by `min` and `max` vectors [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
EMPTY_VOXEL = 0
FILLED = 1
BINARY_VOXELIZATION = True  # Set to False for non-binary mode

def voxelize_triangle(t, t_bbox_grid, p_bbox_grid, unitlength, morton_start, voxels, voxel_data, nfilled):
    # 1. Iterate over the 2D XY grid within the triangle's bounding box
    for x in range(t_bbox_grid.min[0], t_bbox_grid.max[0] + 1):
        for y in range(t_bbox_grid.min[1], t_bbox_grid.max[1] + 1):
            
            # 2. Calculate 2D position in world space
            p_xy = (x * unitlength, y * unitlength)
            
            # 3. XY Plane Containment Test
            inside_xy = True
            for i in range(3):
                v_curr = t.vertices[i]
                v_next = t.vertices[(i + 1) % 3]
                e = (v_next[0] - v_curr[0], v_next[1] - v_curr[1])
                n_xy = (-e[1], e[0])
                d_xy = - (n_xy[0] * v_curr[0] + n_xy[1] * v_curr[1])
                if (n_xy[0] * p_xy[0] + n_xy[1] * p_xy[1]) + d_xy < 0.0:
                    inside_xy = False
                    break
            if not inside_xy:
                continue

            # 4. Determine min and max corners in XY plane for the current voxel
            min_corner = (p_xy[0], p_xy[1], 0.0)
            max_corner = (p_xy[0] + unitlength, p_xy[1] + unitlength, 0.0)
            
            # 5. Swap min/max components if triangle normal components are negative
            if t.normal[0] < 0:
                min_corner[0], max_corner[0] = max_corner[0], min_corner[0]
            if t.normal[1] < 0:
                min_corner[1], max_corner[1] = max_corner[1], min_corner[1]
            
            # 6. Calculate world Z coordinates for min and max corners using plane equation
            # Plane equation: n[X]*x + n[Y]*y + n[Z]*z + d = 0
            denom = -1.0 * t.normal[2]
            if abs(denom) < 1e-9:
                continue  # Skip degenerate case where triangle is parallel to XY plane [S1, unofficial]
            
            z_min_world = (t.normal[0] * min_corner[0] + t.normal[1] * min_corner[1] + t.plane_d) / denom
            z_max_world = (t.normal[0] * max_corner[0] + t.normal[1] * max_corner[1] + t.plane_d) / denom
            
            # 7. Convert world Z to voxel indices
            z_min = int(z_min_world / unitlength)
            z_max = int(z_max_world / unitlength)
            
            # 8. Clamp z_min and z_max to global bounding box grid Z range
            # Fixed: Source code bug clamped z_max using z_min as lower bound; both must be clamped correctly [S1, unofficial]
            z_min = clamp(z_min, p_bbox_grid.min[2], p_bbox_grid.max[2])
            z_max = clamp(z_max, p_bbox_grid.min[2], p_bbox_grid.max[2])
            
            # 9. Handle case where z_min == z_max
            if z_min == z_max:
                index = morton_encode(z_min, y, x)
                if voxels[index - morton_start] != EMPTY_VOXEL:
                    continue
                voxels[index - morton_start] = FILLED
                nfilled += 1
                if not BINARY_VOXELIZATION:
                    avg_color = (t.v0_color + t.v1_color + t.v2_color) / 3.0
                    voxel_data.append(VoxelData(morton=index, color=avg_color, normal=t.normal))
                continue

            # 10. Setup ZX Plane Projection Tests
            n_zx_ei = []
            d_xz_ei = []
            for i in range(3):
                v_curr = t.vertices[i]
                v_next = t.vertices[(i + 1) % 3]
                e = (v_next[0] - v_curr[0], v_next[1] - v_curr[1], v_next[2] - v_curr[2])
                n_zx = (-e[0], e[2])
                if t.normal[1] < 0:
                    n_zx = (-n_zx[0], -n_zx[1])
                n_zx_ei.append(n_zx)
                d_xz = -1.0 * (n_zx[0] * v_curr[2] + n_zx[1] * v_curr[0])
                d_xz += max(0.0, unitlength * n_zx[0])
                d_xz += max(0.0, unitlength * n_zx[1])
                d_xz_ei.append(d_xz)

            # 11. Setup YZ Plane Projection Tests
            n_yz_ei = []
            d_yz_ei = []
            for i in range(3):
                v_curr = t.vertices[i]
                v_next = t.vertices[(i + 1) % 3]
                e = (v_next[0] - v_curr[0], v_next[1] - v_curr[1], v_next[2] - v_curr[2])
                n_yz = (-e[2], e[1])
                if t.normal[0] < 0:
                    n_yz = (-n_yz[0], -n_yz[1])
                n_yz_ei.append(n_yz)
                d_yz = -1.0 * (n_yz[0] * v_curr[1] + n_yz[1] * v_curr[2])
                d_yz += max(0.0, unitlength * n_yz[0])
                d_yz += max(0.0, unitlength * n_yz[1])
                d_yz_ei.append(d_yz)

            # 12. Iterate over Z range
            for z in range(z_min, z_max + 1):
                index = morton_encode(z, y, x)
                if voxels[index - morton_start] != EMPTY_VOXEL:
                    continue
                
                # 13. YZ Projection Test
                p_yz = (y * unitlength, z * unitlength)
                inside_yz = True
                for i in range(3):
                    if (n_yz_ei[i][0] * p_yz[0] + n_yz_ei[i][1] * p_yz[1]) + d_yz_ei[i] < 0.0:
                        inside_yz = False
                        break
                if not inside_yz:
                    continue
                
                # 14. ZX Projection Test
                p_zx = (z * unitlength, x * unitlength)
                inside_zx = True
                for i in range(3):
                    if (n_zx_ei[i][0] * p_zx[0] + n_zx_ei[i][1] * p_zx[1]) + d_xz_ei[i] < 0.0:
                        inside_zx = False
                        break
                if not inside_zx:
                    continue
                
                # 15. Mark voxel as filled
                voxels[index - morton_start] = FILLED
                nfilled += 1
                if not BINARY_VOXELIZATION:
                    avg_color = (t.v0_color + t.v1_color + t.v2_color) / 3.0
                    voxel_data.append(VoxelData(morton=index, color=avg_color, normal=t.normal))

def morton_encode(z, y, x):
    # [G] Standard 3D Morton code bit-interleaving
    m = 0
    for i in range(32):
        m |= (x & (1 << i)) << (2 * i)
        m |= (y & (1 << i)) << (2 * i + 1)
        m |= (z & (1 << i)) << (2 * i + 2)
    return m

def clamp(val, min_val, max_val):
    return max(min_val, min(val, max_val))
```

**Parameters:**
- `unitlength`: The size of a single voxel in world units [S1, unofficial].
- `morton_start`: An offset added to the Morton index to map it to the correct position in the voxels array [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Division by Zero:** The calculation of $z_{min\_world}$ and $z_{max\_world}$ involves division by $-n[Z]$ [S1, unofficial]. If $n[Z]$ is zero, the triangle is parallel to the XY plane, resulting in infinity or NaN [S1, unofficial].
- **Clamping Bug:** The source code contains a bug where $z_{max}$ is clamped using $z_{min}$ as the lower bound argument: `clampval<int>(z_min, p_bbox_grid.min[Z], p_bbox_grid.max[Z])` [S1, unofficial]. The intent is to clamp both $z_{min}$ and $z_{max}$ to the grid bounds, so the correct call should be `clampval<int>(z_max, p_bbox_grid.min[Z], p_bbox_grid.max[Z])` [S1, unofficial].
- **Winding and Normal Orientation:** The flipping of edge normals in ZX and YZ projections depends on the sign of the triangle's normal components ($n[Y]$ and $n[X]$) [S1, unofficial]. Incorrect handling leads to false negatives in containment tests [S1, unofficial].

## Subtopics

### [Huang et al. Voxelization Method](1684-huang-et-al-voxelization-method.md)
This method voxelizes a triangle mesh into a Sparse Voxel DAG by rasterizing triangles into a Morton-coded voxel subgrid using a three-stage overlap test consisting of vertex spheres, edge cylinders, and a plane-barycentric check. It differs from its parent method by operating on a pre-partitioned Morton subgrid range rather than iterating a 2D grid with projection containment tests, allowing for out-of-core processing of large meshes.

## Operating characteristics
- **Time Complexity:** The complexity is proportional to the number of voxels in the triangle's bounding box in the XY plane, multiplied by the depth of the triangle in the Z direction [S1, unofficial]. For each voxel, a constant number of plane tests are performed [S1, unofficial].
- **Memory Complexity:** Depends on the number of filled voxels and the size of the `VoxelData` structure if non-binary voxelization is used [S1, unofficial]. The sparse nature of the DAG allows for efficient storage of empty space [S1, unofficial].
- **Strengths:** Optimized for sparse voxel grids using Morton codes [S1, unofficial]. Explicit projection tests in XY, ZX, and YZ planes provide robust culling [S1, unofficial]. Suitable for out-of-core processing when combined with partitioning [S1, unofficial].
- **Limitations:** Degenerate cases (parallel triangles) require special handling [S1, unofficial]. Potential for bugs in clamping and normal orientation if not carefully implemented [S1, unofficial].
- **When to Use:** When voxelizing large triangle meshes into a sparse structure where memory efficiency is critical and out-of-core processing is beneficial [S1, unofficial].
- **When Not to Use:** For dense voxelizations or when signed distance fields/ray casting are more appropriate for the downstream application [S1, unofficial].

## Background
This method is part of the implementation for "High Resolution Sparse Voxel DAGs" by Zheng et al., as seen in the GitHub repository [leonzheng2/INF584-High_Resolution_Sparse_Voxel_DAGs](https://github.com/leonzheng2/INF584-High_Resolution_Sparse_Voxel_DAGs) [S1, unofficial]. It builds on the concepts of GPU mesh voxelization and extends them with Morton coding and partitioning for scalability [S1, unofficial].

## Sources
[S1] leonzheng2/INF584-High_Resolution_Sparse_Voxel_DAGs, https://github.com/leonzheng2/INF584-High_Resolution_Sparse_Voxel_DAGs, fetched 2026-10-26 (unofficial)
