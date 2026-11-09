# Rasterization-based Voxelization

Also called: Cube Mesh Generation, Hash-based Surface Voxelization, Sparse Voxelization with PositionMap Hashing, Voxelizing a Triangle Mesh, madcad Voxelization

## Overview
Rasterization-based Voxelization converts a continuous triangle mesh into a discrete volumetric representation by projecting the geometry onto a 3D grid using rasterization primitives [S1][S3, S4, unofficial]. It solves the problem of generating a voxelized surface (shell) or point cloud from polygonal data without performing complex surface reconstruction or deep interior filling [S2, S3, unofficial]. It differs from [VoxelGrid from Mesh (Fill Method)](1013-voxelgrid-from-mesh-fill-method.md) by focusing on surface intersection rather than flood-filling the interior, and from [GPU Volume Kernel Voxelization](1496-gpu-volume-kernel-voxelization.md) by typically relying on CPU-side hash-table deduplication and scanline logic rather than parallel fragment shaders [G].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangle mesh defined by vertices $V$ and faces $F$ (indices), a voxel size vector $\mathbf{vs} = (v_x, v_y, v_z)$, a voxel origin $\mathbf{o}$ (typically $[0,0,0]$), and a precision factor $\epsilon$ [S1][S2, S4, unofficial].
*   **Output:** Either a sparse set of occupied voxel coordinates (integer indices $\mathbf{ijk}$), a point cloud of voxel centers with optional colors, or a generated cube mesh where each occupied voxel becomes a unit cube [S1][S4, unofficial].

**Data Structures:**
1.  **Hash Table (Sparse Storage):** Used to store occupied voxels to avoid dense 3D arrays. A common implementation uses a hash function mapping voxel center coordinates to a bucket index, with linked lists handling collisions [S3, S4, unofficial].
    *   *Node Structure:* Contains `position` (voxel center coordinates) and optional `color` [S4, unofficial].
2.  **Output Mesh Structure (`vx_mesh_t`):**
    *   `vertices`: Array of size $N_{voxels} \times 8$, storing the 8 corners of each cube.
    *   `normals`: Array of size 6, storing predefined face normals (18 floats) [S4, unofficial].
    *   `colors`: Array of size $N_{voxels} \times 8$ (or NULL) [S4, unofficial].
    *   `indices`: Array of size $N_{voxels} \times \text{VOXELIZER\_INDICES\_SIZE}$ [S4, unofficial].
    *   `normalindices`: Array of size $N_{voxels} \times \text{VOXELIZER\_INDICES\_SIZE}$ [S4, unofficial].
3.  **Point Cloud Structure (`vx_point_cloud_t`):**
    *   `vertices`: Array of size $N_{voxels}$ [S4, unofficial].
    *   `colors`: Array of size $N_{voxels}$ (or NULL) [S4, unofficial].
    *   `nvertices`: Count of valid points [S4, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Compute the half-voxel size vector $\mathbf{hvs} = \mathbf{vs} \times 0.5$ [S4, unofficial]. Allocate a hash table for storing occupied voxel data [S4, unofficial].
2.  **Rasterization/Intersection:** For each triangle face in the mesh:
    *   Determine the bounding box of the triangle in voxel space.
    *   Iterate through all integer voxel coordinates $(i, j, k)$ within this bounding box.
    *   Test if the triangle intersects the voxel. This can be done by checking if the voxel center lies within the triangle's projection or using a conservative rasterization test [S1][S4, S6, unofficial].
    *   If intersected, compute the voxel center $\mathbf{c} = (i \cdot v_x + o_x + hvs_x, j \cdot v_y + o_y + hvs_y, k \cdot v_z + o_z + hvs_z)$ [S4, unofficial].
    *   Insert $\mathbf{c}$ into the hash table. If the voxel is already present, skip insertion to maintain uniqueness [S4, unofficial].
3.  **Geometry Generation (Optional):**
    *   If generating a cube mesh, iterate through the hash table.
    *   For each unique voxel center $\mathbf{c}$, generate 8 cube vertices relative to $\mathbf{c}$ using $\mathbf{hvs}$:
        $$ \mathbf{v}_{offsets} = \{ (\pm hvs_x, \pm hvs_y, \pm hvs_z) \} $$
    *   Store these vertices in the output mesh array [S4, unofficial].
    *   Assign predefined normals and indices to form the cube faces [S4, unofficial].
4.  **Cleanup:** Free the hash table and return the resulting mesh or point cloud [S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rasterization_voxelization(mesh_vertices, mesh_faces, voxel_size, origin, precision):
    """
    Converts a triangle mesh to a sparse set of occupied voxel centers.
    """
    # 1. Initialize
    hvs = [v * 0.5 for v in voxel_size]
    occupied_voxels = set() # Hash set for O(1) lookup
    
    # 2. Process each triangle
    for face in mesh_faces:
        # Get triangle vertices
        v0 = mesh_vertices[face[0]]
        v1 = mesh_vertices[face[1]]
        v2 = mesh_vertices[face[2]]
        
        # Compute bounding box in voxel space
        # Note: In a full implementation, this involves transforming 
        # triangle bounds to voxel grid coordinates.
        min_i, max_i, min_j, max_j, min_k, max_k = compute_voxel_bounds(v0, v1, v2, voxel_size, origin)
        
        # 3. Iterate over voxels in bounding box
        for i in range(min_i, max_i + 1):
            for j in range(min_j, max_j + 1):
                for k in range(min_k, max_k + 1):
                    # Check intersection
                    # This step varies by implementation: 
                    # - Point Cloud Utils uses a specific rasterization kernel [S1]
                    # - madcad uses PositionMap.hasher.keysfor() [S3, unofficial]
                    # - Generic C lib uses conservative rasterization logic [S4, unofficial]
                    if is_triangle_voxel_intersect(v0, v1, v2, i, j, k, voxel_size, origin, precision):
                        occupied_voxels.add((i, j, k))
                        
    return occupied_voxels

def generate_cube_mesh(occupied_voxels, voxel_size, origin):
    """
    Generates a cube mesh from a set of occupied voxel indices.
    """
    hvs = [v * 0.5 for v in voxel_size]
    vertices = []
    indices = []
    
    # Predefined cube corner offsets relative to center
    offsets = [
        (-hvs[0],  hvs[1],  hvs[2]),
        (-hvs[0], -hvs[1],  hvs[2]),
        ( hvs[0], -hvs[1],  hvs[2]),
        ( hvs[0],  hvs[1],  hvs[2]),
        (-hvs[0],  hvs[1], -hvs[2]),
        (-hvs[0], -hvs[1], -hvs[2]),
        ( hvs[0], -hvs[1], -hvs[2]),
        ( hvs[0],  hvs[1], -hvs[2])
    ]
    
    # Predefined face indices for a cube (6 faces, 2 triangles each, 3 indices per triangle)
    # Standard cube face indices: 0,1,2,3 (front), 4,5,6,7 (back), etc.
    # Simplified for demonstration:
    face_indices = [
        0, 1, 2, 0, 2, 3, # Front
        4, 6, 5, 4, 7, 6, # Back
        0, 4, 5, 0, 5, 1, # Left
        2, 6, 7, 2, 7, 3, # Right
        0, 3, 7, 0, 7, 4, # Top
        1, 5, 6, 1, 6, 2  # Bottom
    ]
    
    base_vertex_index = 0
    for (i, j, k) in occupied_voxels:
        # Calculate center
        center = [
            i * voxel_size[0] + origin[0] + hvs[0],
            j * voxel_size[1] + origin[1] + hvs[1],
            k * voxel_size[2] + origin[2] + hvs[2]
        ]
        
        # Add 8 vertices
        for offset in offsets:
            vertices.append([
                center[0] + offset[0],
                center[1] + offset[1],
                center[2] + offset[2]
            ])
            
        # Add indices
        for idx in face_indices:
            indices.append(base_vertex_index + idx)
            
        base_vertex_index += 8
        
    return vertices, indices
```

**Parameters:**
*   `voxel_size`: Dimensions of a single voxel $(v_x, v_y, v_z)$. Typical values: $0.75, 0.5, 0.2, 0.1$ units [S2, unofficial]. In Point-Cloud-Utils, often $1/256.0$ [S1].
*   `vox_origin`: The world coordinate of the $(0,0,0)$ voxel corner. Typically $[0,0,0]$ [S1].
*   `precision`: A tolerance factor to reduce "holes" artifacts in the surface. Typical value: $0.01$ [S2, S4, unofficial].

**Edge Cases and Pitfalls:**
*   **Holes:** Standard rasterization may miss voxels if triangle edges pass between voxel centers. A precision factor or conservative rasterization is required to mitigate this [S2, S6, unofficial].
*   **Performance:** The output mesh size scales with voxel density. For a torus (441 vertices, 800 faces), a voxel size of $0.75$ results in 61,312 vertices and 91,968 faces; size $0.5$ results in 144,968 vertices and 217,452 faces [S2, unofficial].
*   **Sparse vs Dense:** Using a hash set (sparse) is more memory-efficient for sparse meshes than a dense 3D boolean array [S3, unofficial].

## Subtopics

### [3D Grid Rasterization](903-3d-grid-rasterization.md)
A CPU-based algorithm that determines which voxels in a fixed-resolution 3D grid are intersected by a triangle mesh, producing a binary occupancy grid. It omits face QEF and dual vertex solve steps, focusing solely on the intersect scanline. Choose when CPU-side scanline filling and hash-table deduplication are preferred over GPU parallelization.

### [GPU Volume Kernel Voxelization](1496-gpu-volume-kernel-voxelization.md)
Accelerates voxelization by parallelizing the iteration over the 2D XY plane of the voxel grid using a GPU compute shader. It retains explicit triangle-voxel intersection logic but distributes the workload, achieving significant speedups for high-resolution grids. Choose when CPU performance is a bottleneck.

### [GPU Voxelization via Compute Shaders](2549-gpu-voxelization-via-compute-shaders.md)
Converts meshes to voxel grids by decoupling intersection logic from the fixed-function rasterization pipeline, allowing custom geometric tests and atomic operations on 3D buffers. Choose when custom intersection logic is required beyond standard rasterization.

### [GPU Voxelization](2633-gpu-voxelization.md)
Leverages the GPU's hardware rasterization pipeline to map fragments directly to 3D grid cells, writing directly to a 3D texture UAV or SSBO. Choose for high-throughput rasterization-based approaches.

### [CPU Mesh Voxelization](3137-cpu-mesh-voxelization.md)
Iterates over the bounding box of each triangle and tests each voxel center against the triangle's plane and edge constraints. Outputs a binary occupancy grid or bit-packed table. Choose for exact geometric overlap tests (SAT) rather than simple grid cell intersection.

### [Voxel Rasterization](3204-voxel-rasterization.md)
Focuses on the specific algorithmic steps of iterating over triangle surfaces, interpolating normals/colors, and writing to a voxel buffer, often leveraging GPU acceleration. Choose when explicit sampling along triangle edges and spans is needed.

### [Conservative Voxelization](3247-conservative-voxelization.md)
Guarantees conservative correctness by computing the exact depth range of the intersection between the triangle and the voxel column. Choose when standard rasterization misses intersecting voxels.

### [Conservative Rasterization via Triangle Expansion](3531-conservative-rasterization-via-triangle-expansion.md)
Eliminates voxelization holes by expanding the triangle in clip space and culling the expanded area in the fragment shader. Choose when hardware conservative rasterization is unavailable or insufficient.

### [Multi-view projection](5937-multi-view-projection.md)
Rasterizes geometry from multiple camera viewpoints to solve coverage gaps and geometric distortions inherent in single-view projection. Choose when surface normals are perpendicular to the view direction.

### [Position Map and Image-Space Transfer](6237-position-map-and-image-space-transfer.md)
Projects or transfers information into 2D image space (textures) before reconstructing the 3D volume. Choose when leveraging GPU fixed-function rasterizer or image load/store operations on 2D textures is beneficial for connectivity or real-time performance.

### Rasterization-based Volume Modeling
Builds volume data voxel-by-voxel by instancing primitives, bounding them, and sampling their density function at each overlapping voxel. It "pulls" values into a voxel, contrasting with instantiation-based modeling which "pushes" values. Choose when working with volumetric primitives that extend past their geometric representation.

### Single-Pass Voxelization
Utilizes the geometry shader to select the axis of largest triangle projection for fragment generation, reducing draw calls. Requires conservative rasterization (hardware-supported on Nvidia Maxwell+) and MSAA render targets to prevent incorrectly filling empty cells. Choose for real-time ray-traced voxel global illumination where draw call overhead is critical.

## Operating characteristics
*   **Time Complexity:** $O(N_{triangles} \times N_{voxels\_in\_bbox})$ in the worst case, where $N_{voxels\_in\_bbox}$ is the number of voxels intersecting a triangle's bounding box. With hash-table deduplication, the final output size is proportional to the surface area of the mesh in voxels.
*   **Memory Complexity:** $O(N_{occupied\_voxels})$ for sparse storage. Dense storage would be $O(N_x \times N_y \times N_z)$.
*   **Performance:** The output mesh size can be significantly larger than the input. For example, a torus with 441 vertices and 800 faces can expand to over 140,000 vertices with a voxel size of 0.5 [S2, unofficial]. This impacts rendering performance if not managed [S2, unofficial].
*   **Strengths:** Simple to implement; produces a clean, manifold cube mesh suitable for further processing; handles sparse data efficiently with hash tables [S3, unofficial].
*   **Limitations:** Prone to holes if precision is not handled; output size scales with resolution; CPU-based versions may be slow for high-resolution grids [S1][S2, unofficial].
*   **When to use:** When a discrete, manifold cube mesh representation is needed from a triangle mesh, and the mesh is not excessively dense.
*   **When not to use:** When deep interior voxelization is required (use [Voxel Carving](1213-voxel-carving.md) or [VoxelGrid from Mesh (Fill Method)](1013-voxelgrid-from-mesh-fill-method.md)); when real-time performance on extremely high-resolution grids is critical without GPU acceleration (use [GPU Volume Kernel Voxelization](1496-gpu-volume-kernel-voxelization.md)).

## Background
The method originates from the need to convert polygonal models into voxel-based representations for volume rendering and voxel art. It is implemented in libraries such as Point-Cloud-Utils [S1] and the header-only C voxelizer library [S4, unofficial]. The approach is distinct from geometry shader-based voxelizers, which generate cubes from triangle centroids, by providing more accurate surface intersection [S2, unofficial]. It is also related to rasterization-based volume modeling, which pulls values into voxels by sampling density functions [S5, unofficial].

## Sources
[S1] Voxelizing a Triangle Mesh - Point Cloud Utils, https://fwilliams.info/point-cloud-utils/sections/voxelizing_a_triangle_mesh/, fetched 2026-10-26
[S2] Mesh Voxelization - Geeks3D, https://www.geeks3d.com/20160531/mesh-voxelization/, fetched 2026-10-26 (unofficial)
[S3] In Python, how do I voxelize a 3D mesh - Stack Overflow, https://stackoverflow.com/questions/11851342/in-python-how-do-i-voxelize-a-3d-mesh, fetched 2026-10-26 (unofficial)
[S4] mesh voxelizer library, https://github.com/karimnaaji/voxelizer, fetched 2026-10-26 (unofficial)
[S5] Production Volume Rendering, https://cg.informatik.uni-freiburg.de/intern/seminar/volumeRenderingProductionFundamentals2011.pdf, fetched 2026-10-27 (unofficial)
[S6] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
