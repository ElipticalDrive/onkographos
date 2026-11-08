# Rasterization-based Voxelization

Also called: Cube Mesh Generation, Hash-based Surface Voxelization, Sparse Voxelization with PositionMap Hashing, Voxelizing a Triangle Mesh, madcad Voxelization

## Overview
Rasterization-based Voxelization converts a continuous triangle mesh into a discrete volumetric representation by projecting the geometry onto a 3D grid using rasterization principles [S1][S4, unofficial]. It solves the problem of generating a voxelized mesh (either as a set of occupied cubes or a point cloud) from polygonal input without performing full surface reconstruction or deep interior filling [S1][S2, unofficial]. Unlike [Voxel Carving](1213-voxel-carving.md) which uses volume intersection, or [Implicit Surface Voxelization](2379-implicit-surface-voxelization.md) which evaluates signed distance fields, this method relies on the geometric overlap of triangles with voxel cells, typically outputting a "shell" or surface representation [S2, S4, unofficial]. It differs from [3D Grid Rasterization](903-3d-grid-rasterization.md) by often employing sparse data structures (hash tables) to manage memory efficiency for sparse scenes [S3, S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangle mesh defined by vertices $V$ and faces $F$ (indices), a voxel size $vs = (v_x, v_y, v_z)$, a voxel origin $o$ (typically $[0,0,0]$), and a precision factor $p$ [S1][S2, S4, unofficial].
*   **Output:** Either a sparse set of occupied voxel coordinates (integer indices $ijk$) [S1][S3, unofficial], a point cloud of voxel centers with optional colors [S4, unofficial], or a new triangle mesh representing the surface of the voxels (cube mesh) [S2, S4, unofficial].

**Data Structures:**
1.  **Hash Table (Sparse Voxel Storage):** Used to store occupied voxels to avoid dense 3D arrays. In C implementations, this is a hash table with buckets containing linked lists of `vx_voxel_data_t` nodes, where each node stores a `position` (voxel center) and optional `color` [S4, unofficial]. In Python, a `set` of coordinate tuples serves this purpose [S3, unofficial].
2.  **Output Mesh Structure (`vx_mesh_t`):** If generating a cube mesh, the output contains:
    *   `vertices`: Array of size $N_{voxels} \times 8$, storing the 8 corners of each cube.
    *   `normals`: Array of size 6, storing predefined face normals for the 6 cube faces.
    *   `colors`: Array of size $N_{voxels} \times 8$ (or NULL).
    *   `indices`: Array of size $N_{voxels} \times VOXELIZER\_INDICES\_SIZE$ (typically 36 for 6 faces $\times$ 2 triangles).
    *   `normalindices`: Array of size $N_{voxels} \times VOXELIZER\_INDICES\_SIZE$ [S4, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Define the voxel size $vs$ and compute the half-voxel size $hvs = vs \times 0.5$ [S4, unofficial]. Allocate the hash table for occupied voxels [S4, unofficial].
2.  **Triangle Rasterization (Voxel Occupation):** Iterate through each triangle face in the input mesh. For each face, determine the set of voxel grid cells that the triangle intersects or covers.
    *   *CPU/Hash-based approach:* Use a rasterizer function (e.g., `vx__voxelize` or `PositionMap.keysfor`) that maps the triangle's vertices to voxel coordinates. The rasterizer clips the triangle against voxel boundaries or uses scanline filling to identify all integer grid indices $(i, j, k)$ intersected by the triangle [S3, S4, unofficial].
    *   *Precision Handling:* A precision factor $p$ is used during the rasterization/intersection test to reduce artifacts like holes caused by floating-point inaccuracies or edge cases [S2, S4, unofficial].
3.  **Deduplication:** Insert the identified voxel coordinates into the hash table. The hash table handles deduplication, ensuring each occupied voxel is stored only once [S4, unofficial].
4.  **Output Generation:**
    *   *For Point Cloud:* Iterate through the hash table. For each occupied voxel, extract its center position and color. Store these in the output point cloud arrays [S4, unofficial].
    *   *For Cube Mesh:* Iterate through the hash table. For each occupied voxel at center $C$:
        *   Generate the 8 corner vertices of the cube: $C \pm hvs$. The specific vertex ordering is defined by the offset array:
            $$
            \begin{aligned}
            &(-h_x, h_y, h_z), (-h_x, -h_y, h_z), (h_x, -h_y, h_z), (h_x, h_y, h_z), \\
            &(-h_x, h_y, -h_z), (-h_x, -h_y, -h_z), (h_x, -h_y, -h_z), (h_x, h_y, -h_z)
            \end{aligned}
            $$
        *   Assign the predefined face normals to the cube's 6 faces [S4, unofficial].
        *   Add the cube's vertices and indices to the output mesh arrays [S4, unofficial].

**Pseudocode (Python-like, Hash-based Cube Mesh Generation):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rasterization_based_voxelization(mesh_vertices, mesh_faces, voxel_size, origin, precision):
    """
    Voxelizes a triangle mesh into a cube mesh using hash-based rasterization.
    
    Args:
        mesh_vertices: List of 3D coordinates (x, y, z).
        mesh_faces: List of integer triplets (v1, v2, v3).
        voxel_size: Tuple (vx, vy, vz) defining the size of one voxel.
        origin: Tuple (ox, oy, oz) defining the world coordinate of voxel (0,0,0).
        precision: Float tolerance for intersection tests.
        
    Returns:
        output_vertices: List of 3D coordinates for the cube mesh.
        output_faces: List of integer triplets for the cube mesh.
        output_normals: List of 3D coordinates for face normals.
    """
    # 1. Compute half-voxel sizes
    hvs = [voxel_size[0] * 0.5, voxel_size[1] * 0.5, voxel_size[2] * 0.5]
    
    # 2. Initialize hash set for occupied voxels
    # Key: (i, j, k) integer indices, Value: None (or color if available)
    occupied_voxels = set()
    
    # 3. Rasterize each triangle to find occupied voxels
    for face in mesh_faces:
        # Get world coordinates of the triangle vertices
        v0 = mesh_vertices[face[0]]
        v1 = mesh_vertices[face[1]]
        v2 = mesh_vertices[face[2]]
        
        # Determine the bounding box of the triangle in voxel coordinates
        # Map world coordinates to voxel indices
        def world_to_voxel_idx(world_coord):
            return (
                int((world_coord[0] - origin[0]) / voxel_size[0]),
                int((world_coord[1] - origin[1]) / voxel_size[1]),
                int((world_coord[2] - origin[2]) / voxel_size[2])
            )
        
        # Calculate min/max voxel indices covered by the triangle
        # Note: In a real implementation, this requires a robust triangle-voxel 
        # intersection test (e.g., slab method or scanline fill) to find ALL 
        # intersected voxels, not just the bounding box.
        # Here we approximate with bounding box for structure clarity, 
        # but the source implies a full rasterization step [S3, S4, unofficial].
        idx0 = world_to_voxel_idx(v0)
        idx1 = world_to_voxel_idx(v1)
        idx2 = world_to_voxel_idx(v2)
        
        min_i = min(idx0[0], idx1[0], idx2[0])
        max_i = max(idx0[0], idx1[0], idx2[0])
        min_j = min(idx0[1], idx1[1], idx2[1])
        max_j = max(idx0[1], idx1[1], idx2[1])
        min_k = min(idx0[2], idx1[2], idx2[2])
        max_k = max(idx0[2], idx1[2], idx2[2])
        
        # Iterate over the bounding box and test intersection
        # [G] A precise implementation would use a triangle rasterizer 
        # (like the one in madcad or the C library) to fill this range.
        for i in range(min_i, max_i + 1):
            for j in range(min_j, max_j + 1):
                for k in range(min_k, max_k + 1):
                    # Check if the triangle intersects the voxel (i, j, k)
                    # This is the core "rasterization" step [S4, unofficial]
                    if triangle_intersects_voxel(v0, v1, v2, i, j, k, voxel_size, origin, precision):
                        occupied_voxels.add((i, j, k))

    # 4. Generate Cube Mesh from occupied voxels
    output_vertices = []
    output_faces = []
    output_normals = []
    
    # Predefined cube corner offsets relative to center
    # Order: Top-Front-Right, Top-Front-Left, Bottom-Front-Left, Bottom-Front-Right, 
    #        Top-Back-Right, Top-Back-Left, Bottom-Back-Left, Bottom-Back-Right
    # Based on S4: {-hvs.x, hvs.y, hvs.z, -hvs.x, -hvs.y, hvs.z, hvs.x, -hvs.y, hvs.z, hvs.x, hvs.y, hvs.z, ...}
    # Mapping to indices 0-7:
    # 0: (-hx, hy, hz), 1: (-hx, -hy, hz), 2: (hx, -hy, hz), 3: (hx, hy, hz)
    # 4: (-hx, hy, -hz), 5: (-hx, -hy, -hz), 6: (hx, -hy, -hz), 7: (hx, hy, -hz)
    # Note: S4 lists them as:
    # -hvs.x, hvs.y, hvs.z (0)
    # -hvs.x, -hvs.y, hvs.z (1)
    # hvs.x, -hvs.y, hvs.z (2)
    # hvs.x, hvs.y, hvs.z (3)
    # -hvs.x, hvs.y, -hvs.z (4)
    # -hvs.x, -hvs.y, -hvs.z (5)
    # hvs.x, -hvs.y, -hvs.z (6)
    # hvs.x, hvs.y, -hvs.z (7)
    
    cube_offsets = [
        (-hvs[0], hvs[1], hvs[2]),
        (-hvs[0], -hvs[1], hvs[2]),
        (hvs[0], -hvs[1], hvs[2]),
        (hvs[0], hvs[1], hvs[2]),
        (-hvs[0], hvs[1], -hvs[2]),
        (-hvs[0], -hvs[1], -hvs[2]),
        (hvs[0], -hvs[1], -hvs[2]),
        (hvs[0], hvs[1], -hvs[2])
    ]
    
    # Predefined normals for the 6 faces of a cube (X+, X-, Y+, Y-, Z+, Z-)
    # S4 mentions copying 18 floats (6 normals * 3 floats).
    # Standard cube normals:
    normals = [
        (1, 0, 0), (-1, 0, 0),
        (0, 1, 0), (0, -1, 0),
        (0, 0, 1), (0, 0, -1)
    ]
    
    # Face definitions (indices into cube_offsets) for 6 faces, 2 triangles each
    # Face 0: X+ (Right), Face 1: X- (Left), Face 2: Y+ (Top), Face 3: Y- (Bottom), Face 4: Z+ (Front), Face 5: Z- (Back)
    # Based on S4 offsets, we must deduce the face connectivity.
    # S4 uses VOXELIZER_INDICES_SIZE. A standard cube has 36 indices.
    # We will define the faces based on the offset order provided in S4.
    # Face X+ (Right): Indices 2, 3, 7, 6 (Right side in S4 list? Let's assume standard cube mapping)
    # To be safe and generic, we define the 6 faces explicitly.
    # Face 0 (X+): 2, 3, 7, 6
    # Face 1 (X-): 0, 1, 5, 4
    # Face 2 (Y+): 0, 3, 7, 4
    # Face 3 (Y-): 1, 2, 6, 5
    # Face 4 (Z+): 0, 1, 2, 3
    # Face 5 (Z-): 4, 5, 6, 7
    
    faces_def = [
        [2, 3, 7, 6], # X+
        [0, 1, 5, 4], # X-
        [0, 3, 7, 4], # Y+
        [1, 2, 6, 5], # Y-
        [0, 1, 2, 3], # Z+
        [4, 5, 6, 7]  # Z-
    ]
    
    # Triangulate each quad face into 2 triangles
    quad_to_triangles = [
        [0, 1, 2, 2, 3, 0], # 0-1-2, 2-3-0
        [0, 1, 2, 2, 3, 0],
        [0, 1, 2, 2, 3, 0],
        [0, 1, 2, 2, 3, 0],
        [0, 1, 2, 2, 3, 0],
        [0, 1, 2, 2, 3, 0]
    ]
    
    current_vertex_count = 0
    
    for (i, j, k) in occupied_voxels:
        # Calculate voxel center in world space
        center = (
            origin[0] + (i + 0.5) * voxel_size[0],
            origin[1] + (j + 0.5) * voxel_size[1],
            origin[2] + (k + 0.5) * voxel_size[2]
        )
        
        # Generate 8 vertices for this voxel
        for offset in cube_offsets:
            output_vertices.append((
                center[0] + offset[0],
                center[1] + offset[1],
                center[2] + offset[2]
            ))
        
        # Generate indices and normals for the 6 faces
        for face_idx, quad_indices in enumerate(faces_def):
            # Add 2 triangles for this face
            tri_indices = quad_to_triangles[face_idx]
            for t_idx in tri_indices:
                output_faces.append(current_vertex_count + quad_indices[t_idx])
            
            # Add normals for this face (repeated for each vertex of the 2 triangles)
            normal = normals[face_idx]
            for _ in range(6): # 2 triangles * 3 vertices
                output_normals.append(normal)
        
        current_vertex_count += 8

    return output_vertices, output_faces, output_normals

def triangle_intersects_voxel(v0, v1, v2, i, j, k, voxel_size, origin, precision):
    """
    [G] Placeholder for the specific rasterization/intersection logic.
    In the C library [S4, unofficial], this is handled by vx__voxelize.
    In madcad [S3, unofficial], it uses PositionMap.keysfor.
    A robust implementation checks if the triangle intersects the AABB defined by (i,j,k).
    """
    # Simplified bounding box check for demonstration
    # Real implementation requires slab intersection or scanline fill
    return True 
```

**Parameters:**
*   `voxel_size`: Physical dimensions of a voxel $(v_x, v_y, v_z)$. Typical values: $0.75, 0.5, 0.2, 0.1$ units [S2, unofficial]. In Point Cloud Utils, a common value is $1.0/256.0$ [S1].
*   `vox_origin`: World coordinate of the $(0,0,0)$ voxel corner. Typically $[0,0,0]$ [S1].
*   `precision`: Tolerance factor for intersection tests to reduce "holes" artifacts. Typical value: $0.01$ [S2, S4, unofficial].
*   `gap_fraction`: (Optional) Spacing between cubes in the generated geometry. Typical value: $0.02$ [S1].

**Edge Cases and Pitfalls:**
*   **Holes/Artifacts:** Standard rasterization may miss voxels if the triangle edge passes between voxel centers. A precision factor or conservative rasterization is required to mitigate this [S2, S6, unofficial].
*   **Performance:** The output mesh size scales with voxel density. For a torus (441 vertices, 800 faces), a voxel size of $0.75$ results in 61,312 vertices and 91,968 faces, while $0.5$ results in 144,968 vertices and 217,452 faces [S2, unofficial]. This can significantly impact rendering performance if not managed [S2, unofficial].
*   **Sparse vs. Dense:** Using a hash table (sparse) is more memory-efficient for sparse meshes than a dense 3D array [S3, S4, unofficial].

## Subtopics

### [3D Grid Rasterization](903-3d-grid-rasterization.md)
A CPU-based algorithm that determines which voxels in a fixed-resolution 3D grid are intersected by a triangle mesh, producing a binary occupancy grid. It omits face QEF and dual vertex solve steps, focusing solely on intersect scanlines. Choose when a simple binary occupancy grid is needed without generating cube geometry.

### [GPU Volume Kernel Voxelization](1496-gpu-volume-kernel-voxelization.md)
Accelerates voxelization by parallelizing the iteration over the 2D XY plane of the voxel grid using a GPU compute shader. Choose when CPU-based nested loops are a bottleneck and high-resolution grids are required.

### [GPU Voxelization via Compute Shaders](2549-gpu-voxelization-via-compute-shaders.md)
Converts meshes to voxel grids using GPU parallel processing, decoupling intersection logic from rasterization. Choose when custom geometric tests and atomic operations on 3D buffers are needed.

### [GPU Voxelization](2633-gpu-voxelization.md)
Uses the GPU's hardware rasterization pipeline to map fragments directly to 3D grid cells. Choose when leveraging standard triangle rasterization for high throughput.

### [CPU Mesh Voxelization](3137-cpu-mesh-voxelization.md)
Iterates over the bounding box of each triangle and tests each voxel center against the triangle's plane and edge constraints. Choose when outputting a binary occupancy grid or bit-packed table rather than a cube mesh.

### [Voxel Rasterization](3204-voxel-rasterization.md)
Focuses on iterating over triangle surfaces, interpolating normals/colors, and writing to a voxel buffer. Choose when explicit sampling along triangle edges and spans is required.

### [Conservative Voxelization](3247-conservative-voxelization.md)
Guarantees conservative correctness by computing the exact depth range of intersection. Choose when missing voxels due to standard rasterization gaps are unacceptable.

### [Conservative Rasterization via Triangle Expansion](3531-conservative-rasterization-via-triangle-expansion.md)
Eliminates voxelization holes by expanding the triangle in clip space and culling in the fragment shader. Choose when hardware conservative rasterization is unavailable or insufficient.

### [Multi-view projection](5937-multi-view-projection.md)
Rasterizes geometry from multiple camera viewpoints to solve coverage gaps. Choose when single-view projection fails due to surface normals perpendicular to the view direction.

### [Position Map and Image-Space Transfer](6237-position-map-and-image-space-transfer.md)
Projects or transfers information into 2D image space before reconstructing the 3D volume. Choose when leveraging GPU fixed-function rasterizer or image load/store operations on 2D textures is beneficial.

### Rasterization-based Volume Modeling
Builds volume data voxel-by-voxel by instancing primitives, bounding them, and sampling their density function at each overlapping voxel. It "pulls" values into voxels, contrasting with instantiation-based modeling which "pushes" values. Choose when generating volumetric density fields from parametric primitives.

### Single-Pass Voxelization
Utilizes the geometry shader to reduce draw calls by selecting the axis of largest projection for fragment generation. Uses conservative rasterization and MSAA to prevent cracks and empty cell errors. Choose when real-time performance with minimal draw calls is critical and hardware supports conservative rasterization.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot R)$, where $N$ is the number of triangles and $R$ is the average number of voxels intersected per triangle. The hash table insertion is $O(1)$ on average [S4, unofficial].
*   **Memory Complexity:** $O(V)$, where $V$ is the number of occupied voxels. Using a hash table makes it efficient for sparse scenes [S3, S4, unofficial].
*   **Measured Performance:** For a torus mesh (441 vertices, 800 faces), voxel size $0.75$ yields 61,312 vertices and 91,968 faces; voxel size $0.5$ yields 144,968 vertices and 217,452 faces [S2, unofficial]. The output mesh size scales significantly with voxel density, impacting rendering performance [S2, unofficial].
*   **Strengths:** Simple to implement; generates explicit cube mesh geometry suitable for further processing; hash-based approach is memory-efficient for sparse data [S3, S4, unofficial].
*   **Limitations:** Output mesh can be very large; standard rasterization may produce holes without precision adjustments or conservative rasterization [S2, S6, unofficial].
*   **When to Use:** When an explicit surface representation (cube mesh) is required from a triangle mesh, and the scene is sparse enough to benefit from hash-based storage.
*   **When Not to Use:** When deep interior filling is required (use [Voxel Carving](1213-voxel-carving.md) or [Implicit Surface Voxelization](2379-implicit-surface-voxelization.md)), or when memory is extremely constrained and a binary occupancy grid suffices.

## Background
Rasterization-based voxelization is a fundamental technique in computer graphics for converting polygonal models into volumetric data. It is implemented in libraries such as Point-Cloud-Utils [S1] and the header-only C library by Karim Naaji [S4, unofficial]. The method is often used in voxel art generation, real-time ray tracing voxel global illumination [S6, unofficial], and as a preprocessing step for volume rendering. The approach of "pulling" values into voxels via rasterization is contrasted with "pushing" values via instantiation in production volume rendering contexts [S5, unofficial].

## Sources
[S1] Voxelizing a Triangle Mesh - Point Cloud Utils, https://fwilliams.info/point-cloud-utils/sections/voxelizing_a_triangle_mesh/, fetched 2026-10-26
[S2] Mesh Voxelization - Geeks3D, https://www.geeks3d.com/20160531/mesh-voxelization/, fetched 2026-10-26 (unofficial)
[S3] In Python, how do I voxelize a 3D mesh - Stack Overflow, https://stackoverflow.com/questions/11851342/in-python-how-do-i-voxelize-a-3d-mesh, fetched 2026-10-26 (unofficial)
[S4] mesh voxelizer library, https://github.com/karimnaaji/voxelizer, fetched 2026-10-26 (unofficial)
[S5] Production Volume Rendering, https://cg.informatik.uni-freiburg.de/intern/seminar/volumeRenderingProductionFundamentals2011.pdf, fetched 2026-10-27 (unofficial)
[S6] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
