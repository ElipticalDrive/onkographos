# Rasterization-based Voxelization

Also called: Cube Mesh Generation, Voxelizing a Triangle Mesh

## Overview
Rasterization-based Voxelization converts a continuous triangle mesh into a discrete 3D occupancy grid (voxel array) or a surface mesh of cubes by projecting the geometry onto a 2D plane and using hardware or software rasterization to determine coverage [S1][S2, unofficial][S3, unofficial][S4, unofficial]. It solves the problem of representing complex surface geometry as a volume suitable for ray marching or other volume rendering techniques [G], distinguishing itself from ray-tracing voxelization by leveraging the efficiency of scan-conversion algorithms [S4, unofficial]. Unlike [Interior Voxelize](995-interior-voxelize.md) which fills the volume, this method typically performs "shell voxelization," outputting only the surface voxels [S2, unofficial].

## How it works
The method maps mesh geometry to a fixed-resolution integer lattice [S1][S2, unofficial][S3, unofficial][S6, unofficial]. The core logic involves iterating over triangles, determining which voxels they overlap, and marking those voxels as occupied.

### Inputs and Outputs
**Inputs:**
*   Input mesh vertices ($V$) and faces ($F$).
*   Voxel size ($vs$): dimensions of the target voxel along each axis ($voxelsizex, voxelsizey, voxelsizez$) [S3, unofficial].
*   Voxel origin ($vox\_origin$): the location of the bottom-left-back corner of the $[0, 0, 0]$ voxel [S1].
*   Precision ($p$): a factor to reduce "holes" artifacts during rasterization [S2, unofficial].

**Outputs:**
*   Occupied voxel coordinates ($vox\_ijk$): an array of integer voxel indices [S1].
*   Alternatively, a cube mesh ($cube\_v, cube\_f$) representing the voxelized surface [S1].
*   Alternatively, a point cloud where each point corresponds to a voxel center, optionally with per-voxel colors [S3, unofficial].

### Data Structures
The implementation often uses a hash table to store occupied voxel data to handle sparse grids efficiently [S3, unofficial].

**Hash Table Node (`vx_voxel_data_t`):**
*   `position`: 3D float vector representing the voxel center [S3, unofficial].
*   `color`: 3D float vector (optional) [S3, unofficial].
*   `next`: pointer to the next node in the linked list for collision handling [S3, unofficial].

**Output Mesh (`vx_mesh_t`):**
*   `vertices`: array of size $voxels \times 8$, storing the 8 corners of each cube [S3, unofficial].
*   `normals`: array of size 6, storing predefined face normals [S3, unofficial].
*   `colors`: array of size $voxels \times 8$ (or NULL) [S3, unofficial].
*   `indices`: array of size $voxels \times VOXELIZER\_INDICES\_SIZE$ [S3, unofficial].
*   `normalindices`: array of size $voxels \times VOXELIZER\_INDICES\_SIZE$ [S3, unofficial].
*   `nvertices`, `nindices`, `nnormals`: counters for active elements [S3, unofficial].

### Steps
1.  **Load Mesh:** Load input vertices $v$ and faces $f$ [S1].
2.  **Initialize Parameters:** Define voxel size $vs$ and origin $vox\_origin$ [S1]. Compute half-voxel sizes $hvs = vs \times 0.5$ [S3, unofficial].
3.  **Rasterize:** Call the internal rasterizer to populate a hash table of occupied voxel data. This step iterates over triangles, clips them against voxel boundaries, and inserts voxel centers into the hash table [S3, unofficial].
    *   *Note:* In GPU implementations, this step utilizes the hardware rasterization pipeline (vertex, fragment shaders) and atomic operations for concurrent writes into a 3D texture or buffer [S4, unofficial].
4.  **Generate Geometry (Optional):** If a cube mesh is required, iterate through the hash table. For each occupied voxel, generate the 8 corner vertices relative to the voxel center using the half-voxel sizes [S3, unofficial].
5.  **Finalize:** Free temporary structures (hash table) and return the output mesh or point cloud [S3, unofficial].

### Pseudocode
The following pseudocode implements the CPU-based hash-table approach described in [S3, unofficial], adapted for clarity.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rasterization_based_voxelization(mesh, voxelsizex, voxelsizey, voxelsizez, precision):
    # 1. Initialize voxel size and half-voxel size
    vs = Vector3(voxelsizex, voxelsizey, voxelsizez)
    hvs = vs * 0.5
    
    # 2. Rasterize triangles into a hash table
    # The internal rasterizer handles triangle-voxel intersection
    hash_table = vx__voxelize(mesh, vs, hvs, precision)
    voxel_count = hash_table.size()
    
    # 3. Allocate output mesh arrays
    # Each voxel becomes a cube with 8 vertices
    out_vertices = Array<Vector3>(voxel_count * 8)
    out_normals = Array<Vector3>(6) # Predefined normals for cube faces
    out_colors = Array<Vector3>(voxel_count * 8) if mesh.has_colors else None
    out_indices = Array<uint>(voxel_count * VOXELIZER_INDICES_SIZE)
    out_normal_indices = Array<uint>(voxel_count * VOXELIZER_INDICES_SIZE)
    
    # Copy predefined face normals
    out_normals = copy(vx_normals) # 18 floats (6 normals * 3 components)
    
    # Define the 8 cube corner offsets relative to center
    cube_offsets = [
        Vector3(-hvs.x,  hvs.y,  hvs.z),
        Vector3(-hvs.x, -hvs.y,  hvs.z),
        Vector3( hvs.x, -hvs.y,  hvs.z),
        Vector3( hvs.x,  hvs.y,  hvs.z),
        Vector3(-hvs.x,  hvs.y, -hvs.z),
        Vector3(-hvs.x, -hvs.y, -hvs.z),
        Vector3( hvs.x, -hvs.y, -hvs.z),
        Vector3( hvs.x,  hvs.y, -hvs.z)
    ]
    
    vertex_idx = 0
    index_idx = 0
    
    # 4. Iterate hash table and build cube geometry
    for bucket in hash_table.buckets:
        node = bucket.head
        while node is not None:
            voxel_data = node.data
            center = voxel_data.position
            color = voxel_data.color
            
            # Generate 8 vertices for this voxel's cube
            for i in range(8):
                out_vertices[vertex_idx] = center + cube_offsets[i]
                if out_colors is not None:
                    out_colors[vertex_idx] = color
                vertex_idx += 1
            
            # Generate indices for the cube faces (6 faces, 2 tris each, 3 verts each)
            # Standard cube indexing pattern
            base = vertex_idx - 8
            faces = [
                [0, 1, 2, 0, 2, 3], # Front
                [4, 6, 5, 4, 7, 6], # Back
                [0, 4, 5, 0, 5, 1], # Left
                [2, 6, 7, 2, 7, 3], # Right
                [0, 3, 7, 0, 7, 4], # Top
                [1, 5, 6, 1, 6, 2]  # Bottom
            ]
            
            for face in faces:
                for vert in face:
                    out_indices[index_idx] = base + vert
                    index_idx += 1
            
            node = node.next
            
    # 5. Return output mesh structure
    return Mesh(out_vertices, out_normals, out_colors, out_indices)
```

### Parameters
*   `vox_size`: Typically $1.0 / 256.0$ [S1].
*   `vox_origin`: Typically $[0, 0, 0]$ [S1].
*   `precision`: Typical value $0.01f$ [S2, unofficial].
*   `gap_fraction`: Spacing between cubes in generated geometry, typically $0.02$ [S1].

### Edge Cases and Pitfalls
*   **Holes:** Standard rasterization only covers pixels whose centers lie inside a triangle, leading to small cracks or holes at edges [S5, unofficial]. This is mitigated by the `precision` parameter or by using conservative rasterization [S2, S5, unofficial].
*   **Overfilling:** Conservative rasterization can generate too many fragments, incorrectly filling empty cells [S5, unofficial]. Using MSAA render targets can help solve this [S5, unofficial].
*   **Performance:** The output mesh has significantly more vertices and faces than the input, scaling with voxel density [S2, unofficial]. For a torus mesh (441 vertices, 800 faces), a voxel size of 0.75 results in 61,312 vertices and 91,968 faces [S2, unofficial].

## Subtopics

### [3D Grid Rasterization](903-3d-grid-rasterization.md)
Converts a triangular mesh into a discrete 3D occupancy grid by mapping mesh geometry to a fixed-resolution integer lattice. It differs from direct cube generation by focusing on the occupancy grid itself, suitable for ray marching [G].

### [GPU Volume Kernel Voxelization](1496-gpu-volume-kernel-voxelization.md)
Accelerates voxelization by parallelizing the iteration over the 2D XY plane of the voxel grid using a GPU compute shader. It retains explicit triangle-voxel intersection logic but distributes the workload, achieving significant speedups (e.g., 50x) for high-resolution grids.

### [Implicit Surface Voxelization](2379-implicit-surface-voxelization.md)
Converts a mathematical implicit surface definition into a voxel grid by rasterizing 2D rectangles within a Region of Interest (ROI) and evaluating the implicit function in the pixel shader. It handles complex, non-polygonal geometry directly, unlike triangle-based methods.

### [Chunked UV Rasterization with Face ID Storage](2598-chunked-uv-rasterization-with-face-id-storage.md)
Rasterizes triangles into a 2D texture buffer in UV space, storing the face ID in the alpha channel to enable per-texel face identification and subsequent 3D position interpolation. It decouples rasterization from 3D grid storage, allowing memory-friendly chunked processing of large meshes.

### [GPU Voxelization](2633-gpu-voxelization.md)
Maps 3D triangle meshes to a voxel grid by leveraging the GPU's hardware rasterization pipeline, projecting triangles onto a 2D plane and using shaders to write occupied voxel indices into a buffer. It distinguishes itself by utilizing the GPU rendering pipeline (vertex, geometry, fragment shaders) and atomic operations.

### [CPU Mesh Voxelization](3137-cpu-mesh-voxelization.md)
Converts a triangle mesh into a 3D grid by iterating over the bounding box of each triangle and testing each voxel center against the triangle's plane and edge constraints. It uses exact geometric intersection tests (SAT) rather than projection-based heuristics.

### [Voxel Rasterization](3204-voxel-rasterization.md)
Converts a triangle mesh into a voxelized representation by rendering the geometry into a 3D voxel buffer, supporting both simple occupancy and normal/color mapping. It explicitly iterates over triangle surfaces to calculate interpolated points and normals.

### [Conservative Rasterization via Triangle Expansion](3531-conservative-rasterization-via-triangle-expansion.md)
Eliminates voxelization holes caused by standard rasterization by expanding the triangle in clip space and culling the expanded area in the fragment shader. It implements expansion logic manually in the geometry shader using homogeneous plane mathematics.

### [Multi-view projection](5937-multi-view-projection.md)
Generates a robust 3D voxel representation by rasterizing the geometry from multiple orthogonal viewpoints, typically along the three principal axes. It resolves 3D occupancy by synthesizing data from distinct projection directions, ensuring complete coverage of complex geometry.

### [Position Map and Image-Space Transfer](6237-position-map-and-image-space-transfer.md)
Converts triangle meshes into voxel grids by leveraging standard 2D rasterization hardware to compute surface intersections, generating an intermediate 2D representation (position map or 2D texture worksheets) that is then transferred into the 3D voxel grid. This decouples intersection computation from volume resolution.

### Rasterization-based Volume Modeling
The process of building volume data voxel-by-voxel, considered the third layer in the voxel modeling abstraction hierarchy [S4, unofficial]. It pulls values into a voxel (similar to raytracing), whereas instantiation-based modeling pushes values into voxels [S4, unofficial]. The main loop iterates over voxels in the discrete bounding box, transforms voxel position to world space, samples the primitive, and adds the value to the buffer if positive [S4, unofficial].

### Single-Pass Voxelization
A voxelization approach that utilizes the geometry shader to reduce draw calls by selecting the axis from which each triangle has the largest overall projection for fragment generation [S5, unofficial]. It employs conservative rasterization to generate fragments as long as there is something inside the pixel, not only if its center is covered [S5, unofficial]. Hardware support for conservative rasterization is available on Nvidia Maxwell architecture and later [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of triangles and the number of voxels they overlap. GPU implementations scale with the number of fragments generated [S1][S4, unofficial].
*   **Memory Complexity:** Proportional to the number of occupied voxels. Hash-table based approaches use memory proportional to the number of unique voxels [S3, unofficial].
*   **Performance:** GPU-based methods achieve significant speedups (e.g., 50x) for high-resolution grids compared to CPU-based methods [S1].
*   **Strengths:** Fast, leverages existing hardware rasterization pipelines, suitable for real-time applications [S4, unofficial].
*   **Limitations:** Output mesh size can be very large compared to the input mesh [S2, unofficial]. Standard rasterization can produce holes or cracks [S5, unofficial]. Conservative rasterization can overfill empty cells [S5, unofficial].
*   **When to use:** When real-time performance is required, or when the input is a triangle mesh and a surface voxel representation is needed [S2, S4, unofficial].
*   **When not to use:** When deep voxelization (interior voxels) is required, or when exact geometric intersection tests are preferred over rasterization heuristics [S2, S3, unofficial].

## Background
The technique is implemented in libraries such as Point-Cloud-Utils [S1] and the header-only mesh voxelizer library by Karim Naaji [S3, unofficial]. It is also discussed in the context of production volume rendering [S4, unofficial] and real-time ray-traced voxel global illumination [S5, unofficial]. The method is often compared to geometry shader-based voxelizers, with rasterization-based approaches providing better results due to more accurate surface intersection or rasterization techniques [S2, unofficial].

## Sources
[S1] Voxelizing a Triangle Mesh - Point Cloud Utils, https://fwilliams.info/point-cloud-utils/sections/voxelizing_a_triangle_mesh/, fetched 2026-10-26
[S2] Mesh Voxelization - Geeks3D, https://www.geeks3d.com/20160531/mesh-voxelization/, fetched 2026-10-26 (unofficial)
[S3] mesh voxelizer library, https://github.com/karimnaaji/voxelizer, fetched 2026-10-26 (unofficial)
[S4] Production Volume Rendering, https://cg.informatik.uni-freiburg.de/intern/seminar/volumeRenderingProductionFundamentals2011.pdf, fetched 2026-10-27 (unofficial)
[S5] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
