# Rasterization-based Voxelization

Also called: Cube Mesh Generation, Hash-based Surface Voxelization, Sparse Voxelization with PositionMap Hashing, Voxelizing a Triangle Mesh, madcad Voxelization

## Overview
Rasterization-based Voxelization converts a triangle mesh into a discrete volumetric representation by projecting the geometry onto a 3D grid using hardware rasterization or hash-based spatial mapping, then generating a surface mesh (cube mesh) or point cloud from the occupied voxels [S1][S4, unofficial]. It solves the problem of creating a voxelized surface representation suitable for volume rendering or voxel art, differing from CPU-based geometric intersection methods by leveraging GPU fragment shaders or efficient hash lookups to determine voxel occupancy [S4, unofficial]. Unlike full surface reconstruction methods (like o-voxel) that solve for dual vertices or QEF, this method outputs binary presence or simple cube geometries [S1]. It is distinct from compute-shader-based voxelization by relying on the fixed-function graphics pipeline or image-space transfer rather than manual thread dispatch for intersection logic [S1][S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangle mesh defined by vertices $V$ and faces $F$ (indices) [S1][S2, unofficial].
*   **Parameters:**
    *   `vox_size`: Voxel dimensions $(v_x, v_y, v_z)$ [S4, unofficial]. Typical values range from 0.1 to 1.0 units, or $1/256.0$ in normalized space [S1][S2, unofficial].
    *   `vox_origin`: The world-space coordinate of the bottom-left-back corner of the $(0,0,0)$ voxel [S1]. Typically $[0, 0, 0]$ [S1].
    *   `precision`: A tolerance factor to reduce "holes" artifacts caused by rasterization gaps [S2, unofficial]. Typical value: $0.01$ [S2, unofficial].
    *   `gap_fraction`: The spacing between generated cubes in the output mesh [S1]. Typical value: $0.02$ [S1].
*   **Output:** A voxelized mesh (cube mesh) containing vertices and indices for each occupied voxel, or a point cloud of voxel centers [S1][S4, unofficial].

**Data Structures:**
1.  **Hash Table (Sparse Storage):** Used to store occupied voxel coordinates to avoid dense 3D arrays [S3, S4, unofficial].
    *   Key: Voxel center position or integer grid coordinates $(i, j, k)$ [S3, S4, unofficial].
    *   Value: Voxel data structure containing position and optional color [S4, unofficial].
    *   Implementation: A hash table with buckets, where each bucket contains a linked list of nodes to handle collisions [S4, unofficial].
2.  **Output Mesh (`vx_mesh_t`):**
    *   `vertices`: Array of size $N_{voxels} \times 8$, storing the 8 corners of each cube [S4, unofficial].
    *   `normals`: Array of size 6, storing predefined face normals for the 6 faces of a cube [S4, unofficial].
    *   `colors`: Array of size $N_{voxels} \times 8$ (or NULL) [S4, unofficial].
    *   `indices`: Array of size $N_{voxels} \times \text{VOXELIZER\_INDICES\_SIZE}$ [S4, unofficial].
    *   `normalindices`: Array of size $N_{voxels} \times \text{VOXELIZER\_INDICES\_SIZE}$ [S4, unofficial].

**Steps:**
1.  **Load Mesh:** Read vertices $V$ and faces $F$ from the input file (e.g., OBJ, PLY) [S1][S3, unofficial].
2.  **Compute Half-Voxel Size:** Calculate $hvs = vs \times 0.5$, where $vs = (v_x, v_y, v_z)$ [S4, unofficial].
3.  **Rasterize to Voxels:**
    *   Iterate through each face (triangle) in the mesh [S3, unofficial].
    *   For each triangle, determine the set of voxel coordinates it intersects or covers [S3, S4, unofficial].
    *   *Method A (Hash-based/PositionMap):* Use a `PositionMap` hasher to map triangle points to voxel grid keys. Update a set/hash table with these keys [S3, unofficial].
    *   *Method B (GPU Rasterization):* Project the triangle onto the 3D grid using hardware rasterization. Fragments generated correspond to occupied voxels. Store these voxel coordinates in a hash table [S4, unofficial].
    *   *Precision Handling:* Apply the `precision` parameter during rasterization/intersection to ensure conservative coverage and reduce holes [S2, S4, unofficial].
4.  **Generate Geometry:**
    *   Allocate output mesh arrays based on the number of unique voxels $N_{voxels}$ found in the hash table [S4, unofficial].
    *   Define the 8 corner offsets for a cube centered at the origin relative to $hvs$:
        $$
        \begin{aligned}
        &(-hvs_x, hvs_y, hvs_z), (-hvs_x, -hvs_y, hvs_z), (hvs_x, -hvs_y, hvs_z), (hvs_x, hvs_y, hvs_z), \\
        &(-hvs_x, hvs_y, -hvs_z), (-hvs_x, -hvs_y, -hvs_z), (hvs_x, -hvs_y, -hvs_z), (hvs_x, hvs_y, -hvs_z)
        \end{aligned}
        $$
        [S4, unofficial].
    *   Iterate through the hash table buckets. For each occupied voxel:
        *   Extract the voxel center position $P$ and color $C$ [S4, unofficial].
        *   Compute the 8 world-space vertices by adding $P$ to the corner offsets [S4, unofficial].
        *   Store these vertices in the output mesh array [S4, unofficial].
        *   Assign predefined normals to the cube faces [S4, unofficial].
        *   Generate indices to form the 6 faces of the cube (typically 12 triangles or 36 indices per voxel, or optimized for shared vertices) [S4, unofficial].
5.  **Cleanup:** Free the hash table and intermediate structures [S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rasterization_based_voxelization(mesh, vox_size, vox_origin, precision=0.01, gap_fraction=0.02):
    """
    Converts a triangle mesh to a voxelized cube mesh.
    
    Args:
        mesh: Object with .vertices (N, 3) and .faces (M, 3)
        vox_size: Tuple (vx, vy, vz)
        vox_origin: Tuple (ox, oy, oz)
        precision: Float for hole reduction
        gap_fraction: Float for spacing between cubes
        
    Returns:
        output_vertices: (V, 3)
        output_faces: (F, 3)
    """
    # 1. Compute half-voxel sizes
    hvs = [s * 0.5 for s in vox_size]
    
    # 2. Initialize sparse hash table for occupied voxels
    # Key: (i, j, k) integer coordinates, Value: voxel center position
    occupied_voxels = {}
    
    # 3. Rasterize triangles to voxels
    for face in mesh.faces:
        # Get triangle vertices
        v0 = mesh.vertices[face[0]]
        v1 = mesh.vertices[face[1]]
        v2 = mesh.vertices[face[2]]
        
        # Determine bounding box of triangle in voxel space
        # Transform vertices to voxel grid coordinates
        # Note: Exact rasterization logic depends on implementation (GPU vs CPU Hash)
        # Here we assume a helper function that returns occupied voxel indices
        voxel_indices = rasterize_triangle_to_voxels(v0, v1, v2, vox_size, vox_origin, precision)
        
        for idx in voxel_indices:
            # Calculate voxel center in world space
            center = vox_origin + [idx[0] * vox_size[0] + hvs[0],
                                   idx[1] * vox_size[1] + hvs[1],
                                   idx[2] * vox_size[2] + hvs[2]]
            occupied_voxels[idx] = center

    # 4. Generate cube mesh geometry
    output_vertices = []
    output_faces = []
    
    # Predefined cube corner offsets relative to center
    # Order: Top 4 (z+), Bottom 4 (z-)
    # Indices: 0: -x,+y,+z, 1: -x,-y,+z, 2: +x,-y,+z, 3: +x,+y,+z
    #          4: -x,+y,-z, 5: -x,-y,-z, 6: +x,-y,-z, 7: +x,+y,-z
    corner_offsets = [
        [-hvs[0],  hvs[1],  hvs[2]],
        [-hvs[0], -hvs[1],  hvs[2]],
        [ hvs[0], -hvs[1],  hvs[2]],
        [ hvs[0],  hvs[1],  hvs[2]],
        [-hvs[0],  hvs[1], -hvs[2]],
        [-hvs[0], -hvs[1], -hvs[2]],
        [ hvs[0], -hvs[1], -hvs[2]],
        [ hvs[0],  hvs[1], -hvs[2]]
    ]
    
    # Cube face definitions (indices into the 8 corners)
    # 6 faces, 2 triangles each, 3 indices per triangle
    cube_faces = [
        [0, 1, 2], [0, 2, 3], # Front (approx)
        [4, 6, 5], [4, 7, 6], # Back
        [0, 4, 5], [0, 5, 1], # Left
        [2, 6, 7], [2, 7, 3], # Right
        [1, 5, 7], [1, 7, 3], # Bottom
        [0, 3, 7], [0, 7, 4]  # Top
    ]
    
    vertex_offset = 0
    
    for idx, center in occupied_voxels.items():
        # Generate 8 vertices for this voxel
        voxel_vertices = []
        for offset in corner_offsets:
            voxel_vertices.append([
                center[0] + offset[0],
                center[1] + offset[1],
                center[2] + offset[2]
            ])
        
        # Add vertices to global list
        output_vertices.extend(voxel_vertices)
        
        # Generate faces for this voxel
        for face_indices in cube_faces:
            output_faces.append([
                vertex_offset + face_indices[0],
                vertex_offset + face_indices[1],
                vertex_offset + face_indices[2]
            ])
        
        vertex_offset += 8
        
    return output_vertices, output_faces

def rasterize_triangle_to_voxels(v0, v1, v2, vox_size, vox_origin, precision):
    """
    [G] Placeholder for the specific rasterization logic.
    In GPU implementations, this is handled by the fragment shader.
    In CPU/Hash implementations (like madcad), this uses spatial hashing.
    Returns a list of (i, j, k) voxel indices.
    """
    # Implementation detail omitted: depends on specific library (pcu, madcad, etc.)
    # Must account for precision and bounding box intersection
    pass
```

## Subtopics

### [3D Grid Rasterization](903-3d-grid-rasterization.md)
3D Grid Rasterization converts a triangle mesh into a discrete 3D occupancy grid by projecting the mesh onto three orthogonal planes (XY, YZ, XZ) using hardware rasterization and combining the results. It differs from CPU-based bounding-box intersection methods by leveraging GPU fragment shaders to determine voxel occupancy, and from full surface reconstruction methods (like o-voxel) by outputting only binary presence without solving for dual vertices or QEF. It is a rasterization-based voxelization technique that produces a dense or sparse grid depending on the storage format used.

### [GPU Volume Kernel Voxelization](1496-gpu-volume-kernel-voxelization.md)
GPU Volume Kernel Voxelization is a technique that accelerates the voxelization of triangle meshes by parallelizing the iteration over the 2D XY plane of the voxel grid using a GPU compute shader. It solves the performance bottleneck of CPU-based voxelization, where nested loops over X and Y coordinates are executed sequentially. Unlike Rasterization-based Voxelization and 3D Grid Rasterization which project geometry into 3D textures, this method retains the explicit triangle-voxel intersection logic of CPU implementations but distributes the workload across GPU threads, achieving significant speedups (e.g., 50x) for high-resolution grids.

### [GPU Voxelization via Compute Shaders](2549-gpu-voxelization-via-compute-shaders.md)
GPU Voxelization via Compute Shaders converts 3D triangle meshes into a uniform 3D voxel grid by leveraging GPU parallel processing. It solves the problem of generating volumetric data for real-time ray marching or volume rendering with high throughput. Unlike Rasterization-based Voxelization, which projects triangles via the fixed-function graphics pipeline, compute shader methods decouple intersection logic from rasterization, allowing custom geometric tests and atomic operations on 3D buffers or textures.

### [GPU Voxelization](2633-gpu-voxelization.md)
GPU Voxelization converts a triangle mesh into a regular voxel grid representation by leveraging the GPU's hardware rasterization pipeline. It maps each fragment generated by the rasterizer to a specific cell in the voxel grid and writes data into that cell, typically using a 3D texture bound as an Unordered Access View (UAV) or a Shader Storage Buffer Object (SSBO). Unlike CPU-based ray-tracing methods which are slow due to serial intersection tests, this method parallelizes the conversion across the GPU. It distinguishes itself from its sibling 3D Grid Rasterization by using standard orthographic projection and viewport mapping rather than explicit 3D grid traversal, and from GPU Voxelization via Compute Shaders by relying on the fixed-function rasterizer rather than manual thread dispatch.

### [CPU Mesh Voxelization](3137-cpu-mesh-voxelization.md)
CPU Mesh Voxelization converts a triangle mesh into a 3D grid of voxels by iterating over the bounding box of each triangle and testing each voxel center against the triangle's plane and edge constraints. It differs from its parent Rasterization-based Voxelization by outputting a binary occupancy grid or bit-packed table rather than generating a cube mesh per occupied voxel. It differs from 3D Grid Rasterization by using exact geometric overlap tests (SAT or plane/projection) rather than simple grid cell intersection, and from Voxelization via Triangle Scattering by iterating cells against triangles rather than scattering triangles into cells.

### [Voxel Rasterization](3204-voxel-rasterization.md)
Voxel Rasterization converts a triangle mesh into a 3D voxel grid by projecting triangles onto a 3D volume and marking occupied voxels. It differs from Ray Casting by using fixed-function or compute-shader-style iteration over triangle edges rather than per-voxel ray intersection, and from Marching Cubes by producing a discrete occupancy grid directly rather than a continuous implicit surface. It serves as the core rasterization step in Rasterization-based Voxelization, where it fills a hash table or structured grid with voxel centers and attributes before generating a cube mesh.

### [Conservative Voxelization](3247-conservative-voxelization.md)
Conservative voxelization is a hardware-accelerated algorithm for polygonal models that guarantees conservative correctness: every voxel intersecting the input model is correctly recognized. It differs from standard rasterization-based voxelization by computing the exact depth range of the intersection between the triangle and the voxel column, rather than using a single depth value which leads to missed voxels.

### [Conservative Rasterization via Triangle Expansion](3531-conservative-rasterization-via-triangle-expansion.md)
This method repairs holes in voxelized meshes caused by standard fixed-function rasterization, which only emits fragments for pixels whose centers lie inside the projected triangle. It expands the triangle in clip space within a geometry shader to ensure all potentially covered pixels are rasterized, then culls the overestimated coverage in the fragment shader. It is distinguished from exact bounding polygon methods by using a simple triangle expansion and fragment-shader culling, which is faster in practice despite higher fragment shader workload.

### [Single-Pass Voxelization](4730-single-pass-voxelization.md)
Single-Pass Voxelization is a GPU-based method that converts triangle meshes into a sparse voxel representation in a single rendering pass by leveraging the geometry shader for axis selection and conservative rasterization for coverage. It distinguishes itself from sibling methods by eliminating the need for multiple draw calls per object (via geometry shaders) and avoiding plain grid allocations (via sparse structures), making it suitable for dynamic scenes where objects are added at runtime.

### [Multi-view projection](5937-multi-view-projection.md)
Multi-view projection voxelization generates a volumetric representation of a triangle mesh by rasterizing the geometry from multiple camera viewpoints, typically aligned with the principal axes of the scene's bounding box. It solves the problem of "shadow" voxels and geometric distortion inherent in single-view projections by ensuring every surface element is visible from at least one viewpoint, thereby preventing occlusion artifacts and ensuring 6-connectivity (tunnel-free) coverage. It differs from Single 2D Projection Voxelization by using multiple views to cover all surface normals, and from 3D Grid Rasterization by relying on standard GPU rasterization pipelines rather than explicit 3D grid traversal.

### [Position Map and Image-Space Transfer](6237-position-map-and-image-space-transfer.md)
This family converts 2D rasterized surface data into 3D voxel grids by transferring world-space positions from a 2D texture (the "position map") into a 3D volume texture. It solves the problem of efficiently populating a 3D grid from 2D rasterization outputs, avoiding the computational cost of geometric intersection tests in 3D space. It differs from sibling methods by decoupling the rasterization of geometry (or instances) from the voxelization of the resulting surface points, using image-space coordinates as the bridge.

### Rasterization-based Volume Modeling
Rasterization-based Volume Modeling is the process of building volume data voxel-by-voxel, considered the third layer in the voxel modeling abstraction hierarchy [S5, unofficial]. It involves instancing the primitive representation, bounding it to find the region of voxels it overlaps, and sampling its density function at each voxel in the overlapping region [S5, unofficial]. The main rasterization loop iterates over voxels in the discrete bounding box, transforms voxel position to world space, samples the primitive, and adds the value to the buffer if positive [S5, unofficial]. Rasterization-based modeling pulls values into a voxel (similar to raytracing), whereas instantiation-based modeling pushes values into voxels (similar to micropolygon rendering) [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of triangles $N_t$ and the number of occupied voxels $N_v$. Rasterization is generally $O(N_t \cdot A)$ where $A$ is the area of the triangle in screen/voxel space, followed by $O(N_v)$ for mesh generation [S1][S4, unofficial].
*   **Memory Complexity:** $O(N_v)$ for the hash table and output mesh [S3, S4, unofficial]. Using a hash table makes it memory-efficient for sparse meshes compared to dense 3D arrays [S3, unofficial].
*   **Measured Performance:** For a torus mesh (441 vertices, 800 faces, radius 20 units), a voxel size of 0.75 results in 61,312 vertices and 91,968 faces; a voxel size of 0.5 results in 144,968 vertices and 217,452 faces [S2, unofficial]. For a bunny mesh (2503 vertices, 4968 faces), a voxel size of 0.2 results in 53,720 vertices and 80,580 faces [S2, unofficial].
*   **Strengths:** Fast on GPU due to parallel rasterization; produces clean cube meshes suitable for voxel art; sparse storage handles large scenes efficiently [S1][S3, S4, unofficial].
*   **Limitations:** Output mesh size scales with voxel density, impacting rendering performance if not managed [S2, unofficial]. Standard rasterization may produce "holes" if precision is not handled, requiring conservative rasterization or expansion techniques [S2, unofficial][S1].
*   **When to use:** When a discrete voxel surface representation (cube mesh) is needed for volume rendering or voxel art, and GPU acceleration is available [S1][S2, unofficial].
*   **When not to use:** When deep (solid) voxelization is required, as this method typically performs shell voxelization (surface only) [S2, unofficial]. When exact geometric intersection without rasterization artifacts is critical without conservative techniques [S1].

## Background
Rasterization-based Voxelization originated as a method to leverage GPU hardware for efficient mesh-to-voxel conversion. It is implemented in libraries such as Point-Cloud-Utils [S1] and the C-based voxelizer library by Karim Naaji [S4, unofficial]. The technique is also used in Python via the `madcad` library's `PositionMap` hasher [S3, unofficial]. It is considered a layer in the voxel modeling abstraction hierarchy, specifically for rasterization-based volume modeling [S5, unofficial]. The method is often compared to geometry shader-based voxelization, with rasterization-based approaches generally providing better results due to more accurate surface intersection or rasterization techniques [S2, unofficial].

## Sources
[S1] Voxelizing a Triangle Mesh - Point Cloud Utils, https://fwilliams.info/point-cloud-utils/sections/voxelizing_a_triangle_mesh/, fetched 2026-10-26
[S2] Mesh Voxelization - Geeks3D, https://www.geeks3d.com/20160531/mesh-voxelization/, fetched 2026-10-26 (unofficial)
[S3] In Python, how do I voxelize a 3D mesh - Stack Overflow, https://stackoverflow.com/questions/11851342/in-python-how-do-i-voxelize-a-3d-mesh, fetched 2026-10-26 (unofficial)
[S4] mesh voxelizer library, https://github.com/karimnaaji/voxelizer, fetched 2026-10-26 (unofficial)
[S5] Production Volume Rendering, https://cg.informatik.uni-freiburg.de/intern/seminar/volumeRenderingProductionFundamentals2011.pdf, fetched 2026-10-27 (unofficial)
