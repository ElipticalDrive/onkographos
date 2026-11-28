# GPU Voxelization via Projection and Bitmasking

Also called: hpsoar/voxelizer

## Overview
This method converts a triangle mesh into a 3D voxel grid on the GPU by rasterizing triangles into a bitmask representation [S1, unofficial]. It projects triangles onto 2D planes, tests voxel occupancy against triangle edges using plane equations, and updates a 32-bit integer bitmask per voxel to represent occupancy along the depth axis (Z) [S1, unofficial]. It is distinguished from its sibling [GPU Mesh Voxelization via Plane/Projection Tests](6744-gpu-mesh-voxelization-via-plane-projection-tests.md) by using a compact bitmask per voxel to store depth occupancy, reducing memory usage compared to storing a boolean or float per voxel, which is efficient for sparse or thin geometry [S1, unofficial].

## How it works
**Inputs:**
- Triangle mesh vertices (`vertices`): array of floats, size $3 \times N_{verts}$ [S1, unofficial].
- Triangle mesh indices (`triangles`): array of integers, size $3 \times N_{tris}$ [S1, unofficial].
- Voxel grid parameters:
  - `bbox0`: minimum bounds of the voxel grid (float[3]) [S1, unofficial].
  - `delta`: voxel size (float[3]) [S1, unofficial].
  - `inv_delta`: inverse voxel size (float[3]) [S1, unofficial].
  - `stride`: stride for accessing voxels in memory (int[3]) [S1, unofficial].
  - `vols`: pointer to the voxel grid array (int*) [S1, unofficial].

**Outputs:**
- `vols`: array of integers representing the voxel grid. Each voxel is a 32-bit integer. The bitmask at index $(x, y, z)$ corresponds to the $z$-th bit of the integer at `voxel(x, y, z)` [S1, unofficial].

**Data Structures:**
- `vols`: `int*` array. The layout is linearized 3D. Access to voxel $(x, y, z)$ is calculated as `vols + z * stride[Z] + y * stride[Y] + x * stride[X]` [S1, unofficial].
- `tri_bbox0`, `tri_bbox1`: bounding box of the current triangle in world space (float[3]) [S1, unofficial].

**Steps:**

1.  **Compute Triangle Bounding Box:** For each triangle, determine its axis-aligned bounding box in world space, `tri_bbox0` and `tri_bbox1` [S1, unofficial].

2.  **Map to Voxel Indices:** Calculate the voxel grid range $[start, end]$ for the triangle using the inverse voxel size [S1, unofficial].
    $$ start[i] = (tri\_bbox0[i] - bbox0[i]) \times inv\_delta[i] $$
    $$ end[i] = (tri\_bbox1[i] - bbox0[i]) \times inv\_delta[i] $$
    where $i \in \{X, Y, Z\}$ [S1, unofficial]. These values are floats representing the fractional voxel indices [S1, unofficial].

3.  **Calculate Integer and Bit Indices:** Determine the range of 32-bit integer blocks along the Z-axis [S1, unofficial].
    $$ int\_begin = \lfloor start[Z] \rfloor \gg 5 $$
    $$ bit\_begin = \lfloor start[Z] \rfloor \ \& \ 31 $$
    $$ int\_end = \lfloor end[Z] \rfloor \gg 5 $$
    $$ bit\_end = \lfloor end[Z] \rfloor \ \& \ 31 $$
    Note: The source code uses `start[Z] >> 5` which implies integer division by 32. Since `start` is float, it is cast to int (truncated) before shifting [S1, unofficial].

4.  **Preprocess for Optimization:** Determine the primary axis (X, Y, or Z) for optimization based on the bounding box dimensions [S1, unofficial].
    - If $start[X] == end[X]$, the triangle is degenerate in X [S1, unofficial].
    - If $start[Y] == end[Y]$, the triangle is degenerate in Y [S1, unofficial].
    - If $start[Z] == end[Z]$, the triangle is degenerate in Z [S1, unofficial].
    The preprocessing logic sets an `axis` flag and handles degenerate cases by setting bits directly or returning early [S1, unofficial].

5.  **Iterate 2D Range:** If not degenerate, iterate over the 2D range perpendicular to the primary axis [S1, unofficial]. For a standard case, this involves iterating over $x$ and $y$ within $[start[X], end[X]]$ and $[start[Y], end[Y]]$ [S1, unofficial].

6.  **Projection Test:** For each voxel $(x, y)$ in the 2D range, perform a projection test to determine if the voxel's column intersects the triangle's projection on the XY plane [S1, unofficial].
    - Compute edge vectors: $e_0 = v_1 - v_0$, $e_1 = v_2 - v_1$, $e_2 = v_0 - v_2$ [S1, unofficial].
    - Compute plane distances for the triangle normal $n$ and edges [S1, unofficial].
    - The projection test checks if the point $(x, y)$ is inside the triangle's 2D projection [S1, unofficial]. This is done by evaluating the sign of the cross product or plane equation for each edge [S1, unofficial].
    - Specifically, the code computes `n_d_xy` for each edge, which represents the plane equation parameters for the projection [S1, unofficial].
    - If the projection test fails, skip this voxel [S1, unofficial].

7.  **Compute Bitmask:** If the voxel is inside the triangle's projection, compute the bitmask of occupied voxels along the depth axis (Z) [S1, unofficial].
    - Iterate $z$ from $start[Z]$ to $end[Z]$ [S1, unofficial].
    - For each $z$, check if the point $p = (x, y, z)$ is inside the triangle in 3D [S1, unofficial].
    - The condition is: $(n \cdot p + d_1) \times (n \cdot p + d_2) \le 0$, where $d_1$ and $d_2$ are distances computed from the triangle vertices and normal [S1, unofficial].
    - Additionally, perform projection tests on the YZ and ZX planes to ensure the point is within the triangle's extent in all dimensions [S1, unofficial].
    - If the point is inside, set the $z$-th bit in the bitmask [S1, unofficial].

8.  **Atomic Update:** Atomically OR the computed bitmask into the voxel's integer in the `vols` array [S1, unofficial].
    $$ atomicOr(vol\_ptr, bits) $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
X = 0
Y = 1
Z = 2

def dot3(a, b):
    return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]

def minus3(a, b, c):
    c[0] = a[0] - b[0]
    c[1] = a[1] - b[1]
    c[2] = a[2] - b[2]
    return c

def compute_plane_d(v, delta, x, y, z, n_d_xy):
    n_d_xy[z] = -(n_d_xy[x]*v[x] + n_d_xy[y]*v[y])
    tmp = delta[x] * n_d_xy[x]
    if tmp > 0:
        n_d_xy[z] += tmp
    tmp = delta[y] * n_d_xy[y]
    if tmp > 0:
        n_d_xy[z] += tmp

def compute_plane_n_d_0(v, e, x, y, z, delta, n_d_xy):
    n_d_xy[x] = e[y]
    n_d_xy[y] = -e[x]
    compute_plane_d(v, delta, x, y, z, n_d_xy)

def compute_plane_n_d_1(v, e, x, y, z, delta, n_d_xy):
    n_d_xy[x] = -e[y]
    n_d_xy[y] = e[x]
    compute_plane_d(v, delta, x, y, z, n_d_xy)

def compute_plane_n_d(v, e, x, y, z, n, delta, n_d_xy):
    if n[z] < 0:
        compute_plane_n_d_0(v, e, x, y, z, delta, n_d_xy)
    else:
        compute_plane_n_d_1(v, e, x, y, z, delta, n_d_xy)

def projection_test(p, x, y, z, n_d_xy):
    return n_d_xy[x]*p[x] + n_d_xy[y]*p[y] + n_d_xy[z] >= 0

def process_range(begin, end, p, x, y, z, delta_z, n, d1, d2, n_d_xy, n_d_yz, n_d_zx):
    bits = 0
    for i in range(begin, end):
        p[z] += delta_z
        n_dot_p = dot3(n, p)
        if (n_dot_p + d1) * (n_dot_p + d2) > 0:
            continue
        if projection_test(p, Y, Z, X, n_d_yz) and projection_test(p, Z, X, Y, n_d_zx):
            bits |= (1 << i)
    return bits

def process_simple(int_begin, int_end, bit_begin, bit_end, vol_stride, vols, start, end, axis):
    if int_begin == int_end:
        bits = 0xffffffff if bit_end == 31 else (1 << (bit_end + 1)) - 1
        vol_ptr = vols + int_begin * vol_stride[Z] + start[Y] * vol_stride[Y] + start[X] * vol_stride[X]
        atomicOr(vol_ptr, bits)
    else:
        bits = 0xffffffff if bit_end == 31 else (1 << (bit_end + 1)) - 1
        vol_ptr = vols + int_begin * vol_stride[Z] + start[Y] * vol_stride[Y] + start[X] * vol_stride[X]
        atomicOr(vol_ptr, bits)
        for i in range(int_begin + 1, int_end):
            vol_ptr += vol_stride[Z]
            atomicOr(vol_ptr, 0xffffffff)
        bits = 0xffffffff if bit_end == 31 else (1 << (bit_end + 1)) - 1
        vol_ptr += vol_stride[Z]
        atomicOr(vol_ptr, bits)

def voxelize_kernel(tri_idx, vertices, triangles, N, tri_bbox0, tri_bbox1, bbox0, delta, inv_delta, stride, vols):
    v0 = vertices[3*triangles[tri_idx]:3*triangles[tri_idx]+3]
    v1 = vertices[3*triangles[tri_idx+N]:3*triangles[tri_idx+N]+3]
    v2 = vertices[3*triangles[tri_idx+2*N]:3*triangles[tri_idx+2*N]+3]
    
    e0 = [v1[i]-v0[i] for i in range(3)]
    e1 = [v2[i]-v1[i] for i in range(3)]
    e2 = [v0[i]-v2[i] for i in range(3)]
    edges = [e0, e1, e2]
    
    n = [e0[1]*e1[2]-e0[2]*e1[1], e0[2]*e1[0]-e0[0]*e1[2], e0[0]*e1[1]-e0[1]*e1[0]]
    
    start = [(tri_bbox0[i] - bbox0[i]) * inv_delta[i] for i in range(3)]
    end = [(tri_bbox1[i] - bbox0[i]) * inv_delta[i] for i in range(3)]
    
    int_begin = int(start[Z]) >> 5
    bit_begin = int(start[Z]) & 31
    int_end = int(end[Z]) >> 5
    bit_end = int(end[Z]) & 31
    
    axis = -1
    flag = 0
    if start[X] == end[X]:
        flag += 1
        axis = X
    if start[Y] == end[Y]:
        if flag > 0:
            process_simple(int_begin, int_end, bit_begin, bit_end, stride, vols, start, end, Z)
            return
        flag += 1
        axis = Y
    if start[Z] == end[Z]:
        if flag > 0:
            bits = 1 << bit_begin
            vol = vols + int_begin * stride[Z] + start[Y] * stride[Y] + start[X] * stride[X]
            other_axis = 1 - axis
            for i in range(start[other_axis], end[other_axis] + 1):
                atomicOr(vol, bits)
                vol += stride[other_axis]
            return
        flag += 1
        axis = Z
    
    c = [delta[i] if n[i] > 0 else 0 for i in range(3)]
    d1 = dot3(n, [c[i] - v0[i] for i in range(3)])
    d2 = dot3(n, [delta[i] - c[i] - v0[i] for i in range(3)])
    
    n_d_xy = [[0.0]*3 for _ in range(3)]
    n_d_yz = [[0.0]*3 for _ in range(3)]
    n_d_zx = [[0.0]*3 for _ in range(3)]
    
    for i in range(3):
        compute_plane_n_d(v0, edges[i], X, Y, Z, n, delta, n_d_xy[i])
        compute_plane_n_d(v0, edges[i], Y, Z, X, n, delta, n_d_yz[i])
        compute_plane_n_d(v0, edges[i], Z, X, Y, n, delta, n_d_zx[i])
        
    for x in range(int(start[X]), int(end[X]) + 1):
        for y in range(int(start[Y]), int(end[Y]) + 1):
            p = [x * delta[X] + bbox0[X], y * delta[Y] + bbox0[Y], 0.0]
            inside = True
            for i in range(3):
                if not projection_test(p, X, Y, Z, n_d_xy[i]):
                    inside = False
                    break
            if not inside:
                continue
            
            p_start = [x * delta[X] + bbox0[X], y * delta[Y] + bbox0[Y], start[Z] * delta[Z] + bbox0[Z]]
            vol_ptr = vols + int_begin * stride[Z] + y * stride[Y] + x * stride[X]
            
            bits = process_range(bit_begin, bit_end + 1, p_start, X, Y, Z, delta[Z], n, d1, d2, n_d_xy, n_d_yz, n_d_zx)
            atomicOr(vol_ptr, bits)
```

**Edge Cases and Pitfalls:**
- When `int_begin == int_end`, the triangle spans only one integer block along Z. The bit range is restricted to `[bit_begin, bit_end]` [S1, unofficial].
- When `int_begin != int_end`, the first block uses `[bit_begin, 31]`, middle blocks use `[0, 31]`, and the last block uses `[0, bit_end]` [S1, unofficial].
- Precision issues may arise when triangle vertices lie exactly on voxel boundaries. The use of `>= 0` in projection tests helps, but degenerate triangles (area 0) should be skipped [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N_{tris} \times B_{xy})$, where $B_{xy}$ is the number of voxels in the 2D bounding box of the triangle. The bitmask computation is $O(B_z)$, where $B_z$ is the number of voxels along the Z-axis [S1, unofficial].
- **Memory Complexity:** $O(V)$, where $V$ is the number of voxels in the grid. Each voxel stores a 32-bit integer [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with the number of triangles and the size of their bounding boxes [S1, unofficial].
- **Strengths:** Efficient for sparse or thin geometry due to bitmask representation. Reduces memory usage compared to storing a boolean or float per voxel [S1, unofficial].
- **Limitations:** Limited to 32 voxels per column due to 32-bit integer bitmask. Not suitable for dense geometry where many voxels per column are occupied [S1, unofficial].
- **When to use:** Sparse meshes, thin geometry, real-time voxelization on GPU [S1, unofficial].
- **When not to use:** Dense geometry, large number of voxels per column, need for sub-voxel precision [S1, unofficial].

## Background
This method is implemented in the `hpsoar/voxelizer` repository [S1, unofficial]. It is a GPU-based technique that uses compute shaders to rasterize triangles into a 3D voxel grid. The method is designed for efficiency on modern GPUs, leveraging atomic operations and bitmasking to reduce memory bandwidth and storage requirements [S1, unofficial].

## Sources
[S1] hpsoar/voxelizer, https://github.com/hpsoar/voxelizer, fetched 2026-10-26 (unofficial)
