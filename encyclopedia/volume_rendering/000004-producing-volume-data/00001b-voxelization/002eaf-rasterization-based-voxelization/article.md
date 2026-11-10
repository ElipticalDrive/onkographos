# Rasterization-based Voxelization

Also called: Cube Mesh Generation, Hash-based Surface Voxelization, Sparse Voxelization with PositionMap Hashing, Voxelizing a Triangle Mesh, madcad Voxelization

## Overview
Rasterization-based Voxelization converts a triangle mesh into a discrete volumetric representation by projecting the geometry onto a 3D grid using hardware or software rasterization techniques [S1][S4, S5, unofficial]. It solves the problem of generating surface-level voxel data (shell voxelization) for rendering or simulation by leveraging the efficiency of rasterization pipelines to determine voxel occupancy, avoiding the computational cost of ray-tracing or complex geometric clipping [S1][S3, S5, unofficial]. It differs from [3D Grid Rasterization](903-3d-grid-rasterization.md) by often utilizing sparse hash structures for memory efficiency and from [Conservative Voxelization](3247-conservative-voxelization.md) by potentially relying on standard rasterization which may miss voxels if not augmented with conservative techniques or post-processing [S1][S3, S4, unofficial].

## How it works
The method maps triangle vertices to voxel coordinates, determines which voxels intersect the triangle surface, and outputs either a sparse set of occupied voxel indices or a dense mesh of cubes representing those voxels.

### Inputs and Outputs
*   **Input**: A triangle mesh defined by vertices $V$ and faces $F$ (indices into $V$) [S1][S2, S4, unofficial].
*   **Parameters**:
    *   `vox_size`: Voxel dimensions $(v_x, v_y, v_z)$ [S1][S4, unofficial]. Typical values range from 0.1 to 1.0 units, or $1/256.0$ [S1][S2, unofficial].
    *   `vox_origin`: The world-space coordinate of the $(0,0,0)$ voxel corner, typically $[0,0,0]$ [S1].
    *   `precision`: A tolerance factor to reduce "holes" artifacts in the output mesh, typically $0.01$ [S2, S4, unofficial].
    *   `gap_fraction`: Optional spacing between generated cubes, typically $0.02$ [S1].
*   **Output**:
    *   **Sparse Grid**: A set of integer voxel coordinates $(i,j,k)$ [S1][S3, unofficial].
    *   **Cube Mesh**: A new mesh containing vertices, normals, and indices representing the surface of the occupied voxels [S1][S4, unofficial].
    *   **Point Cloud**: A set of points at voxel centers, optionally with colors [S4, unofficial].

### Data Structures
1.  **Voxel Hash Table**: A hash table mapping voxel positions to data. In C implementations, this stores `vx_voxel_data_t` containing position and color [S4, unofficial]. In Python, a `set` of tuples or a `PositionMap` hasher is used [S3, unofficial].
2.  **Output Mesh Structure** (`vx_mesh_t`):
    *   `vertices`: Array of size $N_{voxels} \times 8$ (8 corners per cube) [S4, unofficial].
    *   `normals`: Array of size 6 (predefined face normals) [S4, unofficial].
    *   `colors`: Array of size $N_{voxels} \times 8$ (optional) [S4, unofficial].
    *   `indices`: Array of size $N_{voxels} \times \text{VOXELIZER\_INDICES\_SIZE}$ [S4, unofficial].
    *   `normalindices`: Array of size $N_{voxels} \times \text{VOXELIZER\_INDICES\_SIZE}$ [S4, unofficial].

### Algorithm Steps
1.  **Initialization**: Load input mesh vertices $V$ and faces $F$ [S1][S2, unofficial]. Define voxel size $vs = (v_x, v_y, v_z)$ and half-voxel size $hvs = vs \times 0.5$ [S4, unofficial].
2.  **Rasterization/Intersection**: For each triangle face $f \in F$:
    *   Determine the bounding box of the triangle in voxel space.
    *   Rasterize the triangle to identify all voxels $(i,j,k)$ that intersect the triangle surface [S3, S4, unofficial].
    *   *Note*: Standard rasterization may miss voxels if triangle centers do not align with voxel centers; conservative rasterization or expansion techniques can mitigate this [S6, unofficial].
    *   Insert occupied voxel coordinates into a hash table or set [S3, S4, unofficial].
3.  **Geometry Generation** (if outputting a mesh):
    *   Allocate output arrays based on the number of unique voxels $N_{voxels}$ [S4, unofficial].
    *   Define the 8 corner offsets for a cube centered at $(0,0,0)$ using $hvs$:
        $$
        \begin{aligned}
        &(-h_x, h_y, h_z), (-h_x, -h_y, h_z), (h_x, -h_y, h_z), (h_x, h_y, h_z), \\
        &(-h_x, h_y, -h_z), (-h_x, -h_y, -h_z), (h_x, -h_y, -h_z), (h_x, h_y, -h_z)
        \end{aligned}
        $$
    *   For each occupied voxel at world position $P$:
        *   Compute the 8 world-space corners by adding the offsets to $P$.
        *   Store vertices and corresponding normals/colors in the output arrays [S4, unofficial].
4.  **Cleanup**: Free temporary hash tables and input structures [S2, S4, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rasterization_voxelization(mesh_vertices, mesh_faces, voxel_size, voxel_origin, precision=0.01):
    """
    Voxelize a triangle mesh using rasterization-based approach.
    Returns a set of occupied voxel coordinates (i,j,k).
    """
    occupied_voxels = set()
    
    # Precompute half voxel size
    hvs = [s * 0.5 for s in voxel_size]
    
    for face in mesh_faces:
        # Get triangle vertices
        v0 = mesh_vertices[face[0]]
        v1 = mesh_vertices[face[1]]
        v2 = mesh_vertices[face[2]]
        
        # Transform vertices to voxel space relative to origin
        # Note: Exact transformation depends on coordinate system conventions
        tv0 = [(v - o) / s for v, o, s in zip(v0, voxel_origin, voxel_size)]
        tv1 = [(v - o) / s for v, o, s in zip(v1, voxel_origin, voxel_size)]
        tv2 = [(v - o) / s for v, o, s in zip(v2, voxel_origin, voxel_size)]
        
        # Determine bounding box in voxel space
        min_v = [min(tv0[i], tv1[i], tv2[i]) for i in range(3)]
        max_v = [max(tv0[i], tv1[i], tv2[i]) for i in range(3)]
        
        # Iterate over integer voxel coordinates in bounding box
        for i in range(floor(min_v[0]), ceil(max_v[0]) + 1):
            for j in range(floor(min_v[1]), ceil(max_v[1]) + 1):
                for k in range(floor(min_v[2]), ceil(max_v[2]) + 1):
                    # Check if voxel (i,j,k) intersects the triangle
                    # This is the core rasterization step.
                    # In GPU implementations, this is done via fragment shaders.
                    # In CPU implementations (like madcad), it uses geometric tests.
                    if voxel_intersects_triangle(i, j, k, tv0, tv1, tv2, precision):
                        occupied_voxels.add((i, j, k))
                        
    return occupied_voxels

def voxel_intersects_triangle(ix, iy, iz, tv0, tv1, tv2, precision):
    """
    Check if the voxel centered at (ix, iy, iz) with size 1 intersects the triangle.
    Uses a conservative test or rasterization logic.
    """
    # Define voxel bounds in voxel space
    # Voxel center is (ix, iy, iz). Extent is +/- 0.5
    # However, rasterization often checks if the triangle covers the voxel center
    # or intersects the voxel volume.
    
    # Simple center test (may have holes without conservative rasterization)
    # A more robust method clips the triangle against the voxel box.
    
    # Placeholder for specific rasterization logic (e.g., barycentric coordinate test)
    # If using madcad's PositionMap, this is handled internally by hasher.keysfor()
    return True # Simplified for structure; real implementation requires geometric intersection
```

### Edge Cases and Pitfalls
*   **Holes**: Standard rasterization only covers pixels/voxels whose centers lie inside the triangle, leading to gaps [S6, unofficial]. Use conservative rasterization or a precision factor to mitigate [S2, S6, unofficial].
*   **Performance**: Output mesh size scales with voxel density. A torus with 441 vertices can produce >60,000 vertices in the voxelized mesh [S2, unofficial].
*   **Sparse vs Dense**: Using a hash table (sparse) is more memory-efficient for sparse meshes than a 3D boolean array [S3, unofficial].

## Subtopics

### [3D Grid Rasterization](903-3d-grid-rasterization.md)
Converts a mesh to a 3D occupancy grid by projecting onto three orthogonal planes (XY, YZ, XZ) using hardware rasterization. It differs from CPU-based methods by leveraging GPU fragment shaders and from surface reconstruction by outputting only binary presence. Choose when a dense or sparse grid is needed without generating cube geometry.

### [GPU Volume Kernel Voxelization](1496-gpu-volume-kernel-voxelization.md)
Accelerates voxelization by parallelizing the iteration over the 2D XY plane of the voxel grid using a GPU compute shader. It retains explicit triangle-voxel intersection logic but distributes it across threads, achieving significant speedups for high-resolution grids. Choose when CPU-based nested loops are a bottleneck.

### [GPU Voxelization via Compute Shaders](2549-gpu-voxelization-via-compute-shaders.md)
Converts meshes to voxel grids using GPU compute shaders, decoupling intersection logic from the fixed-function rasterization pipeline. It allows custom geometric tests and atomic operations on 3D buffers. Choose when custom intersection logic is required beyond standard rasterization.

### [GPU Voxelization](2633-gpu-voxelization.md)
Leverages the GPU's hardware rasterization pipeline to map fragments directly to 3D grid cells, writing directly to a 3D texture UAV or SSBO. It differs from 3D Grid Rasterization by direct 3D texture writes. Choose for high-throughput rasterization-based voxelization on GPU.

### [CPU Mesh Voxelization](3137-cpu-mesh-voxelization.md)
Iterates over the bounding box of each triangle and tests each voxel center against the triangle's plane and edge constraints. It outputs a binary occupancy grid or bit-packed table. Choose for CPU-only implementations requiring exact geometric overlap tests.

### [Voxel Rasterization](3204-voxel-rasterization.md)
Focuses on the specific steps of iterating over triangle surfaces, interpolating attributes, and writing to a voxel buffer, often with GPU acceleration. It explicitly samples along triangle edges and spans. Choose when attribute interpolation (normals, colors) during rasterization is critical.

### [Conservative Voxelization](3247-conservative-voxelization.md)
Guarantees that every voxel intersecting the input model is recognized by computing the exact depth range of intersection. It differs from standard rasterization by avoiding missed voxels. Choose when geometric completeness is required and hardware supports conservative rasterization.

### [Conservative Rasterization via Triangle Expansion](3531-conservative-rasterization-via-triangle-expansion.md)
Eliminates holes by expanding triangles in clip space and culling in the fragment shader, using homogeneous plane mathematics in a geometry shader. Choose when hardware conservative rasterization is unavailable and hole-free coverage is needed.

### [Multi-view projection](5937-multi-view-projection.md)
Rasters geometry from multiple camera viewpoints to solve coverage gaps and distortions of single-view methods. Choose when surface normals are perpendicular to the view direction, causing single-view failures.

### [Position Map and Image-Space Transfer](6237-position-map-and-image-space-transfer.md)
Decouples surface intersection from volume grid population using intermediate 2D image textures (position maps) to store world-space coordinates. Choose when leveraging image-space techniques for efficient mapping to 3D volumes.

### Rasterization-based Volume Modeling
The process of building volume data voxel-by-voxel by instancing primitives, bounding them, and sampling their density function at each overlapping voxel. It "pulls" values into voxels, contrasting with instantiation-based "push" methods. Choose for procedural volume generation from parametric primitives.

### Single-Pass Voxelization
Utilizes a geometry shader to select the axis of largest projection for fragment generation, reducing draw calls. It employs conservative rasterization and MSAA to prevent cracks and empty cell errors. Choose for real-time rendering where draw call overhead is significant and hardware supports conservative rasterization.

## Operating characteristics
*   **Time Complexity**: Depends on the number of triangles $N_t$ and the number of voxels $N_v$ they cover. For rasterization-based methods, it is roughly proportional to the screen/voxel coverage area of the triangles [S1][S4, unofficial].
*   **Memory Complexity**:
    *   Sparse (Hash Table): $O(N_v)$ where $N_v$ is the number of occupied voxels [S3, S4, unofficial].
    *   Dense (3D Array): $O(N_x N_y N_z)$ [S3, unofficial].
    *   Output Mesh: $O(N_v)$ vertices and indices [S4, unofficial].
*   **Performance**: GPU-based methods offer significant speedups (e.g., 50x) over CPU methods for high-resolution grids [S1]. CPU methods are simpler to implement but slower [S2, unofficial].
*   **Strengths**: Efficient use of hardware rasterization; generates clean surface meshes; sparse storage saves memory.
*   **Limitations**: Standard rasterization may produce holes; output mesh size can be large; requires careful handling of precision to avoid artifacts [S2, S6, unofficial].
*   **When to Use**: When a surface-level voxel representation (cube mesh) is needed for rendering or further processing, and the input is a triangle mesh.
*   **When Not to Use**: When deep (solid) voxelization is required, or when the input is not a triangle mesh (e.g., point clouds, implicit surfaces).

## Background
Rasterization-based voxelization originated from the need to efficiently convert polygonal models into voxel grids for real-time rendering and volume visualization. Early implementations used CPU-based geometric tests, but the advent of GPU hardware rasterization allowed for parallelized projection of triangles into 3D textures or hash tables [S1][S2, S4, unofficial]. Libraries like Point-Cloud-Utils and the voxelizer library by Karim Naaji popularized these techniques for generating cube meshes from voxels [S1][S4, unofficial]. The method is widely used in voxel art, real-time ray tracing, and volume rendering pipelines where surface geometry must be discretized [S1][S5, S6, unofficial].

## Sources
[S1] Voxelizing a Triangle Mesh - Point Cloud Utils, https://fwilliams.info/point-cloud-utils/sections/voxelizing_a_triangle_mesh/, fetched 2026-10-26
[S2] Mesh Voxelization - Geeks3D, https://www.geeks3d.com/20160531/mesh-voxelization/, fetched 2026-10-26 (unofficial)
[S3] In Python, how do I voxelize a 3D mesh - Stack Overflow, https://stackoverflow.com/questions/11851342/in-python-how-do-i-voxelize-a-3d-mesh, fetched 2026-10-26 (unofficial)
[S4] mesh voxelizer library, https://github.com/karimnaaji/voxelizer, fetched 2026-10-26 (unofficial)
[S5] Production Volume Rendering, https://cg.informatik.uni-freiburg.de/intern/seminar/volumeRenderingProductionFundamentals2011.pdf, fetched 2026-10-27 (unofficial)
[S6] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
