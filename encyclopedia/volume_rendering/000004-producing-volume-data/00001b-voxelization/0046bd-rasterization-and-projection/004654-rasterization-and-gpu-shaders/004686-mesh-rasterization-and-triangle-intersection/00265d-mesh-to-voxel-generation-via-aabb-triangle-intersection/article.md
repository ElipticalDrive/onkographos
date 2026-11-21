# Mesh to Voxel Generation via AABB-Triangle Intersection

Also called: MeshToVoxel, Triangle Mesh Voxelization, Triangle Voxelization via AABB-Triangle SAT, Triangle-to-Voxel Conversion, Voxelizer

## Overview
This method converts a triangle mesh into a solid 3D voxel grid by iterating over the Axis-Aligned Bounding Box (AABB) of each triangle and testing for intersection with every voxel cube within that box using the Separating Axis Theorem (SAT) [S2, S4, unofficial]. It differs from [Template-based Voxelization](7964-template-based-voxelization.md) by using explicit geometric intersection tests rather than precomputed templates, and from [Fake Voxelization via Triangle-Cube Intersection](6951-fake-voxelization-via-triangle-cube-intersection.md) by typically being followed by a scanline fill to produce a solid interior rather than just a surface shell [S1, S3, unofficial]. It builds on [Mesh Rasterization and Triangle Intersection](11783-mesh-rasterization-and-triangle-intersection.md) by applying the intersection logic to a discrete volumetric domain.

## How it works
**Inputs and Outputs:**
*   **Input:** A triangle mesh defined by vertices $V$ and indices $I$, a world-space transformation matrix $M$, a voxel size $s$ (float), and a bounding box for the output grid.
*   **Output:** A 3D occupancy grid (voxel array) where each cell is marked as empty or solid.

**Data Structures:**
*   `Triangle`: Struct containing three `Vector3` fields ($p_1, p_2, p_3$) representing vertices in world space [S4, unofficial].
*   `_voxels`: A 3D integer array `int[y, z, x]` with dimensions derived from the bounding box and voxel size. Values are `0` (empty), `1` (intersected/shell), or `2` (occupied/filled) [S4, unofficial]. Alternatively, a `HashSet<UVec3>` can be used to store unique voxel coordinates [S2, unofficial].
*   `Bounds`: A bounding box defined by `min` and `max` vectors in world space [S4, unofficial].

**Steps:**

1.  **Grid Initialization:**
    *   Transform all mesh vertices to world space using $M$ [S2, S4, unofficial].
    *   Compute the world-space bounding box of the transformed vertices by finding the component-wise minimum ($v_{min}$) and maximum ($v_{max}$) of all vertices [S2, S4, unofficial].
    *   Round $v_{min}$ down to the nearest multiple of $s$ and $v_{max}$ up to the nearest multiple of $s$ [S4, unofficial].
    *   If any dimension of the bounding box is zero (i.e., $max_i == min_i$), add $s$ to $max_i$ to ensure non-zero grid dimensions [S4, unofficial].
    *   Calculate the grid dimensions $(N_x, N_y, N_z)$ as $\lceil (max - min) / s \rceil$ [S1, S4, unofficial].
    *   Initialize the `_voxels` array with zeros [S4, unofficial].

2.  **Shell Generation (Surface Voxelization):**
    *   Iterate through each triangle in the mesh [S2, S4, unofficial].
    *   Compute the triangle's bounding box in world space [S2, unofficial].
    *   Determine the range of voxel indices $[x_{min}, x_{max}] \times [y_{min}, y_{max}] \times [z_{min}, z_{max}]$ that overlap with the triangle's bounding box. This is done by translating the bounding box min by subtracting the grid origin and dividing by $s$, then flooring/ceiling [S2, unofficial].
    *   For each voxel index $(ix, iy, iz)$ in this range:
        *   Compute the voxel center in world space:
            $$ c = (ix \cdot s + s/2, iy \cdot s + s/2, iz \cdot s + s/2) + bounds.min $$
            [S4, unofficial]
        *   Perform an **AABB-Triangle Intersection Test** using the Separating Axis Theorem (SAT) [S4, unofficial].
            *   Translate triangle vertices relative to the voxel center: $v_i \leftarrow v_i - c$ [S2, unofficial].
            *   Compute edge vectors: $e_1 = v_2 - v_1$, $e_2 = v_3 - v_2$, $e_3 = v_1 - v_3$ [S4, unofficial].
            *   Compute the triangle normal: $n_{tri} = e_1 \times e_2$ [S2, unofficial].
            *   Define AABB axes: $u_0=(1,0,0)$, $u_1=(0,1,0)$, $u_2=(0,0,1)$ [S2, unofficial].
            *   Test separation along 13 axes:
                1.  **AABB Face Normals ($u_0, u_1, u_2$):**
                    *   Project triangle vertices onto axis $u$: $p_i = v_i \cdot u$ for $i \in \{0,1,2\}$ [S2, unofficial].
                    *   Compute the radius of the AABB projection: $r = s/2 \cdot (|u \cdot u_0| + |u \cdot u_1| + |u \cdot u_2|)$ [S2, unofficial]. Note: Since $u$ is a basis vector, $r = s/2$.
                    *   Separation condition: $\max(\min(p_0,p_1,p_2), -\max(p_0,p_1,p_2)) > r$ [S2, unofficial]. If true, the axis separates the triangle from the AABB.
                2.  **Edge Cross Products ($u_k \times e_j$):**
                    *   Compute 9 cross-product axes $a_{kj} = u_k \times e_j$ for $k,j \in \{0,1,2\}$ [S2, unofficial].
                    *   For each axis $a$, compute the projection interval of the triangle: $[min_p, max_p]$ where $min_p = \min(v_i \cdot a)$ and $max_p = \max(v_i \cdot a)$ [S4, unofficial].
                    *   Compute the radius of the AABB projection: $r_a = s/2 \cdot (|a \cdot u_0| + |a \cdot u_1| + |a \cdot u_2|)$ [S4, unofficial].
                    *   Separation condition: $min_p > r_a$ or $max_p < -r_a$ [S4, unofficial].
                3.  **Triangle Normal ($n_{tri}$):**
                    *   Project triangle vertices onto $n_{tri}$: $p_i = v_i \cdot n_{tri}$ [S2, unofficial].
                    *   Compute the radius of the AABB projection along $n_{tri}$: $r_n = s/2 \cdot (|n_{tri} \cdot u_0| + |n_{tri} \cdot u_1| + |n_{tri} \cdot u_2|)$ [S2, unofficial].
                    *   Separation condition: $\max(\min(p_0,p_1,p_2), -\max(p_0,p_1,p_2)) > r_n$ [S2, unofficial].
            *   If all 13 axes pass the SAT test (no separation), the triangle intersects the voxel [S2, unofficial].
        *   If an intersection is found, mark `_voxels[iy, iz, ix] = 1` and break the triangle loop for this voxel (optimization: one intersection is sufficient to mark the voxel as part of the shell) [S4, unofficial].

3.  **Solid Fill (Scanline Fill):**
    *   Process the voxel grid slice by slice (e.g., along the Z-axis), and each slice row by row (e.g., along the X-axis) [S1, S3, unofficial].
    *   Initialize a 'pen' state to 'Up' (not filling) for each row [S1, S3, unofficial].
    *   For each voxel in the row:
        *   If the voxel is empty (`_voxels == 0`), continue to the next voxel without changing the pen state [S1, unofficial].
        *   If the voxel is part of the shell (`_voxels == 1`):
            *   Cast a horizontal ray from left to right through the center of the voxel to find all intersecting triangles [S1, S3, unofficial].
            *   Identify the 'last' triangle $T_{last}$ intersected by the ray (the one with the largest intersection parameter or furthest in the ray direction) [S1, S3, unofficial].
            *   Determine the orientation of $T_{last}$ relative to the scan direction (X-axis). Let $n_x$ be the x-component of the normal of $T_{last}$ [S3, unofficial].
            *   **Pen State Update:**
                *   If $n_x < 0$ (normal faces left, i.e., outward from the model's interior relative to the scan direction), set Pen = Down [S1, S3, unofficial].
                *   If $n_x > 0$ (normal faces right, i.e., inward or exiting), set Pen = Up [S1, S3, unofficial].
            *   Mark the current voxel as solid (`_voxels = 2`) if the pen is Down [S3, unofficial].
        *   If the pen is Down, mark subsequent empty voxels as solid (`_voxels = 2`) as the scan moves to the right [S1, S3, unofficial].
        *   If the pen is Up, leave empty voxels empty [S1, S3, unofficial].
    *   Repeat for every row in the slice, and every slice in the volume [S1, S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mesh_to_voxel(mesh, transform, voxel_size):
    # 1. Grid Initialization
    vertices_world = [transform * v for v in mesh.vertices]
    min_bound = min(vertices_world, key=lambda v: v.x) # Component-wise min
    max_bound = max(vertices_world, key=lambda v: v.x) # Component-wise max
    
    # Round down min, round up max
    min_bound = floor(min_bound / voxel_size) * voxel_size
    max_bound = ceil(max_bound / voxel_size) * voxel_size
    
    # Ensure non-zero size
    for i in range(3):
        if max_bound[i] == min_bound[i]:
            max_bound[i] += voxel_size
            
    bounds_min = min_bound
    bounds_max = max_bound
    bounds_size = bounds_max - bounds_min
    
    # Grid dimensions
    Nx = int(ceil(bounds_size.x / voxel_size))
    Ny = int(ceil(bounds_size.y / voxel_size))
    Nz = int(ceil(bounds_size.z / voxel_size))
    
    # Initialize voxel array (0: empty, 1: shell, 2: filled)
    voxels = [[[0 for _ in range(Nx)] for _ in range(Nz)] for _ in range(Ny)]
    
    # 2. Shell Generation
    for triangle in mesh.triangles:
        # Transform triangle vertices
        v1, v2, v3 = [transform * p for p in triangle.vertices]
        
        # Compute triangle bounding box
        t_min = min(v1, v2, v3, key=lambda p: p.x) # Component-wise
        t_max = max(v1, v2, v3, key=lambda p: p.x) # Component-wise
        
        # Compute voxel index range
        x_min = int(floor((t_min.x - bounds_min.x) / voxel_size))
        x_max = int(ceil((t_max.x - bounds_min.x) / voxel_size))
        y_min = int(floor((t_min.y - bounds_min.y) / voxel_size))
        y_max = int(ceil((t_max.y - bounds_min.y) / voxel_size))
        z_min = int(floor((t_min.z - bounds_min.z) / voxel_size))
        z_max = int(ceil((t_max.z - bounds_min.z) / voxel_size))
        
        # Clamp to grid bounds
        x_min = max(0, x_min)
        x_max = min(Nx - 1, x_max)
        y_min = max(0, y_min)
        y_max = min(Ny - 1, y_max)
        z_min = max(0, z_min)
        z_max = min(Nz - 1, z_max)
        
        for ix in range(x_min, x_max + 1):
            for iy in range(y_min, y_max + 1):
                for iz in range(z_min, z_max + 1):
                    # Voxel center
                    center = Vector3(
                        (ix + 0.5) * voxel_size + bounds_min.x,
                        (iy + 0.5) * voxel_size + bounds_min.y,
                        (iz + 0.5) * voxel_size + bounds_min.z
                    )
                    
                    if aabb_triangle_intersect(center, voxel_size, v1, v2, v3):
                        voxels[iy][iz][ix] = 1
                        break # Optimization: one intersection is enough for shell
    
    # 3. Solid Fill
    for iz in range(Nz):
        for iy in range(Ny):
            pen_down = False
            for ix in range(Nx):
                if voxels[iy][iz][ix] == 0:
                    if pen_down:
                        voxels[iy][iz][ix] = 2
                    continue
                
                if voxels[iy][iz][ix] == 1:
                    # Ray cast through voxel center along X-axis
                    ray_origin = Vector3(
                        bounds_min.x + ix * voxel_size + voxel_size/2,
                        bounds_min.y + iy * voxel_size + voxel_size/2,
                        bounds_min.z + iz * voxel_size + voxel_size/2
                    )
                    ray_direction = Vector3(1, 0, 0)
                    
                    # Find last intersecting triangle
                    last_triangle = None
                    max_t = -infinity
                    
                    for triangle in mesh.triangles:
                        v1, v2, v3 = [transform * p for p in triangle.vertices]
                        t = ray_triangle_intersect(ray_origin, ray_direction, v1, v2, v3)
                        if t is not None and t > max_t:
                            max_t = t
                            last_triangle = triangle
                    
                    if last_triangle:
                        # Determine pen state based on normal of last triangle
                        n = cross(last_triangle.v2 - last_triangle.v1, last_triangle.v3 - last_triangle.v1)
                        if n.x < 0:
                            pen_down = True
                        else:
                            pen_down = False
                    
                    # Mark current voxel as filled if pen is down
                    if pen_down:
                        voxels[iy][iz][ix] = 2
                    else:
                        voxels[iy][iz][ix] = 2 # Shell is also solid
                else:
                    # Should not happen if logic is correct, but handle case
                    pass
                    
    return voxels

def aabb_triangle_intersect(center, size, v1, v2, v3):
    # Translate vertices relative to box center
    v1 -= center
    v2 -= center
    v3 -= center
    
    # Edge vectors
    e1 = v2 - v1
    e2 = v3 - v2
    e3 = v1 - v3
    
    # Triangle normal
    n_tri = cross(e1, e2)
    
    # AABB axes
    u0 = Vector3(1, 0, 0)
    u1 = Vector3(0, 1, 0)
    u2 = Vector3(0, 0, 1)
    
    axes = [u0, u1, u2, 
            cross(u0, e1), cross(u0, e2), cross(u0, e3),
            cross(u1, e1), cross(u1, e2), cross(u1, e3),
            cross(u2, e1), cross(u2, e2), cross(u2, e3),
            n_tri]
            
    half_size = size / 2.0
    
    for axis in axes:
        # Project triangle vertices
        p0 = dot(v1, axis)
        p1 = dot(v2, axis)
        p2 = dot(v3, axis)
        
        min_p = min(p0, p1, p2)
        max_p = max(p0, p1, p2)
        
        # Project AABB
        # Radius is half_size * sum of absolute components of axis in AABB basis
        # Since u0, u1, u2 are basis, this is half_size * (|axis.x| + |axis.y| + |axis.z|)
        r = half_size * (abs(axis.x) + abs(axis.y) + abs(axis.z))
        
        # Separation test
        if min_p > r or max_p < -r:
            return False
            
    return True

def ray_triangle_intersect(origin, direction, v1, v2, v3):
    # Möller-Trumbore algorithm
    e1 = v2 - v1
    e2 = v3 - v1
    h = cross(direction, e2)
    a = dot(e1, h)
    if abs(a) < 1e-8:
        return None
    f = 1.0 / a
    s = origin - v1
    u = f * dot(s, h)
    if u < 0.0 or u > 1.0:
        return None
    q = cross(s, e1)
    v = f * dot(direction, q)
    if v < 0.0 or u + v > 1.0:
        return None
    t = f * dot(e2, q)
    if t > 1e-8:
        return t
    return None
```

## Subtopics

### [Level 1 Triangle-Voxel Classification](4872-level-1-triangle-voxel-classification.md)
Level 1 Triangle-Voxel Classification classifies input triangles into a coarse 3D voxel grid by mapping vertices to voxel indices and testing overlap with all candidate voxels the triangle spans. It identifies candidate voxels via vertex mapping rather than testing every voxel against every triangle, enabling parallel processing where each thread handles one triangle.

### [Tessellation-based Voxelization](5430-tessellation-based-voxelization.md)
Tessellation-based Voxelization is a GPU-accelerated method for generating 3D voxel grids from triangle meshes by leveraging the hardware tessellation pipeline to subdivide geometry before rasterization. It differs from fixed-function or CPU-based voxelizers by enabling adaptive triangle density and displacement mapping during the generation phase, allowing for higher fidelity in complex regions.

### [Fake Voxelization via Triangle-Cube Intersection](6951-fake-voxelization-via-triangle-cube-intersection.md)
Fake Voxelization via Triangle-Cube Intersection is a surface-only voxelization technique that digitizes a triangulated mesh by independently testing each triangle against a grid of axis-aligned cubes (voxels). Unlike solid voxelization methods that fill the interior volume using Constructive Solid Geometry (CSG), this approach outputs only the voxels intersected by the mesh surface, making it computationally simpler and suitable for generating surface representations. It is distinguished from its siblings by its independence from tessellation depth and its reliance on a template-based intersection test to control topological separability (6- or 26-connectivity).

### [Robust Mesh Voxelization via Exact Predicates](8770-robust-mesh-voxelization-via-exact-predicates.md)
A mesh-native voxelizer that produces an occupancy grid or a signed-distance field (SDF) over a Piecewise Linear Complex (PLC)'s bounding box, using exact-predicate classification to avoid epsilon-ray speckle. Unlike [Level 1 Triangle-Voxel Classification](4872-level-1-triangle-voxel-classification.md) which may rely on approximate ray casting, this method uses Shewchuk's exact predicates for geometric calculations to ensure robust classification of watertight meshes in single precision. It differs from [Tessellation-based Voxelization](5430-tessellation-based-voxelization.md) by operating directly on the input triangle mesh without prior subdivision, and from [Fake Voxelization via Triangle-Cube Intersection](6951-fake-voxelization-via-triangle-cube-intersection.md) by providing true inside/outside parity checks rather than mere intersection counts.

### [Triangle Subdivision for Voxelization](11480-triangle-subdivision-for-voxelization.md)
Triangle Subdivision for Voxelization is a pre-processing method that recursively splits mesh triangles to ensure they are small enough to be voxelized efficiently and predictably. It solves the problem of variable voxel counts per triangle, which complicates memory allocation and parallel processing in out-of-core or batch-based voxelization pipelines. Unlike general-purpose mesh smoothing or tessellation, this method is geometry-driven, targeting specific edge-length or area thresholds relative to the voxel resolution to guarantee homogeneous voxel generation and compact Morton index ranges. It differs from [Level 1 Triangle-Voxel Classification](4872-level-1-triangle-voxel-classification.md) by actively modifying the mesh geometry rather than just classifying existing triangles, and from [Tessellation-based Voxelization](5430-tessellation-based-voxelization.md) by using a deterministic, CPU/GPU-friendly iterative queue rather than hardware tessellation pipelines.

### [Voxel Plane Intersection via Vertex Weights](15504-voxel-plane-intersection-via-vertex-weights.md)
This method determines which voxels within a triangle's Axis-Aligned Bounding Box (AABB) are intersected by the triangle's plane and its edge tangent planes. It serves as the final rasterization step in mesh-to-voxel generation, refining the candidate voxels identified by coarse AABB tests. It is distinguished from sibling methods by its reliance on a discrete vertex-weight summation algorithm rather than continuous geometric intersection tests or tessellation, offering a brute-force but highly parallelizable approach suitable for CPU implementation.

### Mesh-based VCP Grid Generation
Mesh-based VCP Grid Generation computes a Vector-to-Closest-Point (VCP) from each grid point to the mesh triangles using barycentric coordinates of the perpendicular projection [S5, unofficial]. If the projection is outside the triangle, it computes the closest point on the edges [S5, unofficial]. Triangles are inserted into an empty grid, updating grid points if they are empty or point to a triangle with a greater VCP distance [S5, unofficial]. The VCP data is then expanded using region growing, where grid points adopt the triangle of their neighbors if it results in a shorter VCP distance [S5, unofficial].

### Exact Voxelization Method
An alternative voxelization method implemented in binvox, written by Eric Haines, using triangle overlap code by Tomas Akenine-Möller [S6]. A bug existed in version 1.17 where triangles completely at max x/y/z were not voxelized correctly [S6]. In version 1.02, the -e option caused triangles to be voxelized too thick [S6]. When using the -e method with .obj to .schematic conversion, the -mb parameter can be used to specify block IDs in material names [S6].

## Operating characteristics
*   **Time Complexity:** $O(N_t \cdot N_v)$ in the worst case, where $N_t$ is the number of triangles and $N_v$ is the number of voxels in the bounding box of each triangle. With the SAT test, each intersection check is constant time but involves 13 axis tests. The scanline fill is $O(N_{voxels})$.
*   **Memory Complexity:** $O(N_x \cdot N_y \cdot N_z)$ for the voxel grid.
*   **Performance:** The method is straightforward to implement and parallelize (e.g., per-voxel or per-triangle parallelism) [S4, unofficial]. However, it can be slow for high-resolution grids or complex meshes due to the large number of intersection tests.
*   **Strengths:** Produces solid, watertight voxel models suitable for physics simulations [S1, unofficial]. The SAT test is robust and handles arbitrary triangle orientations.
*   **Limitations:** The brute-force intersection testing can be computationally expensive. The scanline fill requires ray casting for each shell voxel, which adds overhead. Precision issues may arise with floating-point arithmetic in the SAT test, though exact predicates can mitigate this [S2, unofficial].
*   **When to use:** When a solid, volumetric representation is required for simulation (e.g., heat diffusion, fracture) [S1, unofficial]. When the mesh is relatively simple or the resolution is low.
*   **When not to use:** When only a surface representation is needed (use [Fake Voxelization via Triangle-Cube Intersection](6951-fake-voxelization-via-triangle-cube-intersection.md)). When performance is critical and the mesh is very complex (consider [Tessellation-based Voxelization](5430-tessellation-based-voxelization.md) or [Robust Mesh Voxelization via Exact Predicates](8770-robust-mesh-voxelization-via-exact-predicates.md)).

## Background
The method was popularized by David Rosen in a 2009 blog post describing voxelization for automatic skinning and physical simulations in games [S1, S3, unofficial]. It is based on the principle of solidifying a surface shell and then filling the interior using a scanline algorithm. The AABB-Triangle intersection test using the Separating Axis Theorem is a standard technique in computer graphics for collision detection and has been adapted for voxelization [S4, unofficial]. The method is used in various game development contexts for creating destructible environments and physics bodies [S1, unofficial].

## Sources
[S1] Triangle mesh voxelization, http://blog.wolfire.com/2009/11/Triangle-mesh-voxelization, fetched 2026-10-26 (unofficial)
[S2] mkmarek/svo-rs, https://github.com/mkmarek/svo-rs, fetched 2026-10-26 (unofficial)
[S3] Triangle Mesh Voxelization - Game Developer, https://www.gamedeveloper.com/programming/triangle-mesh-voxelization, fetched 2026-10-26 (unofficial)
[S4] MeshからVoxel生成, https://qiita.com/fukaken5050/items/cba76aa593372dc135b7, fetched 2026-10-27 (unofficial)
[S5] Vector-to-Closest-Point Octree for Surface Ray-Casting, https://doi.org/10.2312/vmv.20151259, fetched 2026-10-26 (unofficial)
[S6] binvox 3D mesh voxelizer, keywords: voxelization, voxelisation, 3D model, https://www.patrickmin.com/binvox/, fetched 2026-10-29
