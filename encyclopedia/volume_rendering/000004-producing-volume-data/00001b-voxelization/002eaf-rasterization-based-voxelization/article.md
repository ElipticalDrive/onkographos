# Rasterization-based Voxelization

Also called: Cube Mesh Generation, Hash-based Surface Voxelization, Sparse Voxelization with PositionMap Hashing, Voxelizing a Triangle Mesh, madcad Voxelization

## Overview
Rasterization-based Voxelization converts a continuous triangle mesh into a discrete volumetric representation by projecting the mesh onto a 3D grid using graphics pipeline primitives or CPU-side scanline algorithms [S1][S3, S5, unofficial]. It solves the problem of generating a surface-only (shell) voxel grid or a mesh of voxel cubes from polygonal input, avoiding the computational cost of deep interior filling or surface reconstruction [S2, S5, unofficial]. It differs from [Voxel Carving](1213-voxel-carving.md) by using constructive rasterization rather than destructive view-projection, and from [Analytic and Algebraic Methods](7913-analytic-and-algebraic-methods.md) by relying on discrete sampling of triangle bounds rather than exact implicit function evaluation [S5, unofficial].

## How it works
The method maps continuous triangle vertices to discrete voxel indices, determines occupancy via rasterization logic, and optionally generates explicit cube geometry for each occupied voxel.

### Inputs and Outputs
*   **Inputs**:
    *   Input mesh: Array of vertices $V$ and faces $F$ (triangles) [S1][S4, unofficial].
    *   Voxel size: $v_s = (v_{sx}, v_{sy}, v_{sz})$, the physical dimensions of a voxel [S4, unofficial]. Typical values: $0.75, 0.5, 0.2, 0.1$ units [S2, unofficial].
    *   Voxel origin: $v_o = (v_{ox}, v_{oy}, v_{oz})$, the world coordinate of the $(0,0,0)$ voxel corner [S1]. Typical value: $[0, 0, 0]$ [S1].
    *   Precision: $p$, a tolerance factor to reduce "holes" artifacts in the surface [S2, unofficial]. Typical value: $0.01f$ [S2, unofficial].
    *   Gap fraction: $g$, the spacing between cubes in the generated geometry [S1]. Typical value: $0.02$ [S1].
*   **Outputs**:
    *   Occupancy Grid: Array of integer voxel coordinates $(i, j, k)$ [S1].
    *   Voxel Mesh: A new mesh where each occupied voxel is represented by a cube [S2, S4, unofficial].
    *   Point Cloud: A set of points at voxel centers, optionally with colors [S4, unofficial].

### Data Structures
1.  **Hash Table (Sparse Storage)**: Used to store occupied voxel coordinates, avoiding dense 3D arrays [S3, S4, unofficial].
    *   Key: Voxel center position or integer index.
    *   Value: `voxel_data_t` containing position and optional color [S4, unofficial].
    *   Implementation: Linked-list buckets or Python `set` [S3, S4, unofficial].
2.  **Output Mesh Layout (`vx_mesh_t`)** [S4, unofficial]:
    *   `vertices`: Array of size $N_{voxels} \times 8$.
    *   `normals`: Array of size $6$ (predefined face normals).
    *   `colors`: Array of size $N_{voxels} \times 8$ (optional).
    *   `indices`: Array of size $N_{voxels} \times VOXELIZER\_INDICES\_SIZE$.
    *   `normalindices`: Array of size $N_{voxels} \times VOXELIZER\_INDICES\_SIZE$.

### Algorithm Steps
1.  **Initialization**:
    *   Compute half-voxel size: $hvs = v_s \times 0.5$ [S4, unofficial].
    *   Initialize an empty hash table `table` for occupied voxels [S4, unofficial].
    *   Initialize voxel counter $N_{voxels} = 0$ [S4, unofficial].

2.  **Rasterization (Triangle Processing)**:
    *   For each triangle face $f$ in the input mesh:
        *   Determine the bounding box of the triangle in world space.
        *   Map the bounding box to voxel grid indices.
        *   Iterate over all voxel cells $(i, j, k)$ within this bounding box.
        *   Test if the triangle intersects the voxel cell. This can be done via:
            *   CPU-side scanline filling and hash-table deduplication [S1].
            *   GPU fragment shaders with conservative rasterization [S6, unofficial].
            *   Exact geometric overlap tests (SAT or plane/projection) [S3, S5, unofficial].
        *   If intersected, compute the voxel center $c = (i \cdot v_{sx} + v_{ox} + hvs_x, \dots)$ [S4, unofficial].
        *   Insert $c$ into `table`. If the voxel was not already present, increment $N_{voxels}$ [S4, unofficial].

3.  **Geometry Generation (Optional)**:
    *   Allocate output mesh arrays based on $N_{voxels}$ [S4, unofficial].
    *   Define 8 cube corner offsets relative to center:
        $$
        \begin{aligned}
        &(-hvs_x, hvs_y, hvs_z), (-hvs_x, -hvs_y, hvs_z), (hvs_x, -hvs_y, hvs_z), (hvs_x, hvs_y, hvs_z), \\
        &(-hvs_x, hvs_y, -hvs_z), (-hvs_x, -hvs_y, -hvs_z), (hvs_x, -hvs_y, -hvs_z), (hvs_x, hvs_y, -hvs_z)
        \end{aligned}
        $$ [S4, unofficial].
    *   For each non-empty bucket in `table`:
        *   Extract voxel position $pos$ and color $col$ [S4, unofficial].
        *   Generate 8 vertices for the cube by adding offsets to $pos$ [S4, unofficial].
        *   Add indices to form the cube faces (typically 12 triangles or 6 quads) [S4, unofficial].
        *   Copy predefined face normals to the output mesh [S4, unofficial].
    *   If generating a point cloud, store $pos$ and $col$ in the point cloud arrays [S4, unofficial].

4.  **Cleanup**:
    *   Free the hash table and temporary structures [S4, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rasterization_based_voxelization(vertices, faces, voxel_size, voxel_origin, precision=0.01, gap_fraction=0.02):
    """
    Voxelize a triangle mesh into a sparse voxel grid and optionally generate cube geometry.
    
    Args:
        vertices: List of 3D coordinates (Nx3).
        faces: List of triangle indices (Mx3).
        voxel_size: Tuple (sx, sy, sz) of float voxel dimensions.
        voxel_origin: Tuple (ox, oy, oz) of float world coordinates for voxel (0,0,0).
        precision: Float tolerance for intersection tests.
        gap_fraction: Float spacing between generated cubes.
        
    Returns:
        occupied_voxels: Set of (i, j, k) integer tuples.
        voxel_mesh: Optional mesh data (vertices, indices) if requested.
    """
    # 1. Initialization
    hvs = (voxel_size[0] * 0.5, voxel_size[1] * 0.5, voxel_size[2] * 0.5)
    occupied_voxels = set() # Sparse hash table
    
    # 2. Rasterization
    for face in faces:
        # Get triangle vertices
        v0 = vertices[face[0]]
        v1 = vertices[face[1]]
        v2 = vertices[face[2]]
        
        # Compute bounding box in world space
        min_x = min(v0[0], v1[0], v2[0])
        max_x = max(v0[0], v1[0], v2[0])
        min_y = min(v0[1], v1[1], v2[1])
        max_y = max(v0[1], v1[1], v2[1])
        min_z = min(v0[2], v1[2], v2[2])
        max_z = max(v0[2], v1[2], v2[2])
        
        # Map to voxel grid indices
        min_ix = floor((min_x - voxel_origin[0]) / voxel_size[0])
        max_ix = floor((max_x - voxel_origin[0]) / voxel_size[0])
        min_iy = floor((min_y - voxel_origin[1]) / voxel_size[1])
        max_iy = floor((max_y - voxel_origin[1]) / voxel_size[1])
        min_iz = floor((min_z - voxel_origin[2]) / voxel_size[2])
        max_iz = floor((max_z - voxel_origin[2]) / voxel_size[2])
        
        # Iterate over voxels in bounding box
        for i in range(min_ix, max_ix + 1):
            for j in range(min_iy, max_iy + 1):
                for k in range(min_iz, max_iz + 1):
                    # Check intersection of triangle with voxel cell
                    # Note: Exact intersection test depends on implementation (CPU scanline, GPU fragment, SAT)
                    # [G] A simple conservative test: check if triangle center is within voxel bounds expanded by precision
                    # or use a robust triangle-box intersection algorithm.
                    if triangle_intersects_voxel(v0, v1, v2, i, j, k, voxel_origin, voxel_size, precision):
                        occupied_voxels.add((i, j, k))
                        
    # 3. Geometry Generation (Optional)
    voxel_mesh = None
    if generate_mesh:
        cube_vertices = []
        cube_indices = []
        vertex_offset = 0
        
        # Predefined cube corner offsets relative to center
        offsets = [
            (-hvs[0], hvs[1], hvs[2]), (-hvs[0], -hvs[1], hvs[2]), (hvs[0], -hvs[1], hvs[2]), (hvs[0], hvs[1], hvs[2]),
            (-hvs[0], hvs[1], -hvs[2]), (-hvs[0], -hvs[1], -hvs[2]), (hvs[0], -hvs[1], -hvs[2]), (hvs[0], hvs[1], -hvs[2])
        ]
        
        # Cube face indices (12 triangles)
        faces_indices = [
            0, 1, 2, 0, 2, 3,  # Front
            4, 6, 5, 4, 7, 6,  # Back
            0, 4, 5, 0, 5, 1,  # Left
            2, 6, 7, 2, 7, 3,  # Right
            0, 3, 7, 0, 7, 4,  # Top
            1, 5, 6, 1, 6, 2   # Bottom
        ]
        
        for (i, j, k) in occupied_voxels:
            # Compute voxel center
            cx = voxel_origin[0] + i * voxel_size[0] + hvs[0]
            cy = voxel_origin[1] + j * voxel_size[1] + hvs[1]
            cz = voxel_origin[2] + k * voxel_size[2] + hvs[2]
            
            # Generate 8 vertices
            for dx, dy, dz in offsets:
                cube_vertices.append((cx + dx, cy + dy, cz + dz))
            
            # Generate indices
            for idx in faces_indices:
                cube_indices.append(vertex_offset + idx)
                
            vertex_offset += 8
            
        voxel_mesh = (cube_vertices, cube_indices)
        
    return occupied_voxels, voxel_mesh

def triangle_intersects_voxel(v0, v1, v2, i, j, k, origin, size, precision):
    """
    [G] Placeholder for specific intersection logic.
    Common implementations include:
    - CPU: Scanline fill or Separating Axis Theorem (SAT).
    - GPU: Conservative rasterization.
    - Simple: Check if any vertex is inside, or if bounding boxes intersect with tolerance.
    """
    # Simplified check for pseudocode completeness:
    # Check if triangle bounding box intersects voxel bounding box
    min_x = min(v0[0], v1[0], v2[0])
    max_x = max(v0[0], v1[0], v2[0])
    min_y = min(v0[1], v1[1], v2[1])
    max_y = max(v0[1], v1[1], v2[1])
    min_z = min(v0[2], v1[2], v2[2])
    max_z = max(v0[2], v1[2], v2[2])
    
    voxel_min_x = origin[0] + i * size[0]
    voxel_max_x = origin[0] + (i + 1) * size[0]
    voxel_min_y = origin[1] + j * size[1]
    voxel_max_y = origin[1] + (j + 1) * size[1]
    voxel_min_z = origin[2] + k * size[2]
    voxel_max_z = origin[2] + (k + 1) * size[2]
    
    # Expand voxel bounds by precision for conservative test
    p = precision
    return (max_x >= voxel_min_x - p and min_x <= voxel_max_x + p and
            max_y >= voxel_min_y - p and min_y <= voxel_max_y + p and
            max_z >= voxel_min_z - p and min_z <= voxel_max_z + p)
```

### Edge Cases and Pitfalls
*   **Holes**: Standard rasterization may miss voxels if triangle centers fall outside voxel centers. Use a precision parameter or conservative rasterization to mitigate [S2, S6, unofficial].
*   **Performance**: The output mesh size scales with voxel density. A torus with 441 vertices and 800 faces can result in 61,312 vertices at voxel size 0.75, or 144,968 at 0.5 [S2, unofficial].
*   **Sparse vs Dense**: Using a hash table or set is more memory-efficient for sparse meshes than a dense 3D boolean array [S3, S4, unofficial].

## Subtopics

### [3D Grid Rasterization](903-3d-grid-rasterization.md)
A CPU-based algorithm that determines which voxels in a fixed-resolution 3D grid are intersected by a triangle mesh, producing a binary occupancy grid or a grid with interpolated colors. It solves the problem of converting continuous polygonal geometry into a discrete volumetric representation without performing surface reconstruction (such as QEF or dual vertex solving). It differs from full o-voxel implementations by omitting the face QEF, boundary QEF, and dual vertex solve steps, focusing solely on the intersect scanline to find occupied voxels. It differs from GPU-based siblings like [GPU Volume Kernel Voxelization](1496-gpu-volume-kernel-voxelization.md) by relying on CPU-side scanline filling and hash-table deduplication rather than parallel fragment shaders.

### [GPU Volume Kernel Voxelization](1496-gpu-volume-kernel-voxelization.md)
GPU Volume Kernel Voxelization is a technique that accelerates the voxelization of triangle meshes by parallelizing the iteration over the 2D XY plane of the voxel grid using a GPU compute shader. It solves the performance bottleneck of CPU-based voxelization, where nested loops over X and Y coordinates are executed sequentially. Unlike rasterization-based alternatives that project geometry into 3D textures, this method retains the explicit triangle-voxel intersection logic of CPU implementations but distributes the workload across GPU threads, achieving significant speedups (e.g., 50x) for high-resolution grids.

### [Implicit Surface Voxelization](2379-implicit-surface-voxelization.md)
Implicit Surface Voxelization converts a mathematical implicit surface definition into a discrete voxel grid by rasterizing 2D rectangles within a Region of Interest (ROI) and evaluating the implicit function in the pixel shader. It differs from triangle-based rasterization methods (e.g., [Rasterization-based Voxelization](908-rasterization-based-voxelization.md)) by operating directly on analytical functions rather than polygonal approximations, allowing for the handling of complex, non-polygonal geometry.

### [GPU Voxelization via Compute Shaders](2549-gpu-voxelization-via-compute-shaders.md)
GPU Voxelization via Compute Shaders converts 3D triangle meshes into a uniform 3D voxel grid by leveraging GPU parallel processing. It solves the problem of generating volumetric data for real-time ray marching or volume rendering with high throughput. Unlike [Rasterization-based Voxelization](908-rasterization-based-voxelization.md), which projects triangles via the fixed-function graphics pipeline, compute shader methods decouple intersection logic from rasterization, allowing custom geometric tests and atomic operations on 3D buffers or textures.

### [GPU Voxelization](2633-gpu-voxelization.md)
GPU Voxelization converts a triangle mesh into a regular voxel grid by leveraging the GPU's hardware rasterization pipeline to map fragments directly to 3D grid cells. It solves the performance bottleneck of CPU-based ray-tracing voxelization by parallelizing the intersection test across the rasterizer. It differs from [3D Grid Rasterization](903-3d-grid-rasterization.md) by using a 3D texture UAV or SSBO as the direct write target rather than a 2D accumulation buffer, and from [GPU Volume Kernel Voxelization](1496-gpu-volume-kernel-voxelization.md) by relying on standard triangle rasterization rather than volume kernels.

### [Mesh-to-Voxel Geometry Shader](2800-mesh-to-voxel-geometry-shader.md)
The Mesh-to-Voxel Geometry Shader is a real-time DirectX technique that converts mesh objects into voxel-like cubes by iterating over integer grid positions within a triangle's bounding box and emitting explicit cube geometry for grid cells close to the triangle plane. Unlike rasterization-based voxelization which fills pixels or conservative voxelization which determines occupancy, this method explicitly generates the 3D geometry of every intersecting voxel, resulting in significantly higher vertex and triangle counts.

### [CPU Mesh Voxelization](3137-cpu-mesh-voxelization.md)
CPU Mesh Voxelization converts a triangle mesh into a 3D grid of voxels by iterating over the bounding box of each triangle and testing each voxel center against the triangle's plane and edge constraints. It differs from its parent [Rasterization-based Voxelization](908-rasterization-based-voxelization.md) by explicitly rasterizing triangles into a hash table of voxel centers and colors, then generating a cube mesh for each occupied voxel, whereas CPU Mesh Voxelization typically outputs a binary occupancy grid or bit-packed table. It differs from [3D Grid Rasterization](903-3d-grid-rasterization.md) by using exact geometric overlap tests (SAT or plane/projection) rather than simple grid cell intersection, and from [Voxelization via Triangle Scattering](3779-voxelization-via-triangle-scattering.md) by iterating cells against triangles rather than scattering triangles into cells.

### [Voxel Rasterization](3204-voxel-rasterization.md)
Voxel Rasterization converts a triangle mesh into a voxelized representation by rendering the geometry into a 3D voxel buffer, supporting both simple occupancy and normal/color mapping. It differs from its sibling [3D Grid Rasterization](903-3d-grid-rasterization.md) by explicitly iterating over triangle surfaces to calculate interpolated points and normals rather than relying on a pure grid traversal, and from [GPU Volume Kernel Voxelization](1496-gpu-volume-kernel-voxelization.md) by utilizing a Direct3D 11-based rendering pipeline or CUDA kernels for the rasterization step.

### [Conservative Voxelization](3247-conservative-voxelization.md)
Conservative voxelization is a hardware-accelerated algorithm for polygonal models that guarantees conservative correctness: every voxel intersecting the input model is correctly recognized, solving the problem where previous approaches miss intersecting voxels. It differs from standard rasterization-based voxelization [Rasterization-based Voxelization] by computing the exact depth range of the intersection between the triangle and the voxel column, rather than using a single depth value (e.g., the fragment center) which leads to missed voxels.

### [Conservative Rasterization via Triangle Expansion](3531-conservative-rasterization-via-triangle-expansion.md)
This method eliminates voxelization holes caused by standard rasterization, which only covers pixels whose centers lie inside a triangle, by expanding the triangle in clip space and culling the expanded area in the fragment shader. It differs from sibling methods like [Conservative Voxelization](3247-conservative-voxelization.md) by implementing the expansion logic manually in the geometry shader using homogeneous plane mathematics, rather than relying on hardware-specific conservative rasterization extensions or post-processing hole-filling.

### [Multi-view projection](5937-multi-view-projection.md)
Multi-view projection voxelization converts a triangle mesh into a voxel grid by rasterizing the geometry from multiple camera viewpoints, typically aligned with the principal axes of the scene's bounding box. This approach solves the coverage gaps and geometric distortions inherent in single-view projection methods, which fail when surface normals are perpendicular to the view direction. It differs from single-view methods by ensuring comprehensive coverage through redundancy, and from compute-shader-based methods by leveraging standard graphics pipeline rasterization.

### [Parametric Surface Voxelization](6032-parametric-surface-voxelization.md)
Parametric Surface Voxelization converts a parametric surface (such as a subdivision surface or implicit field) into a voxel grid by rasterizing points in the 2D parameter space rather than the 3D object space. It differs from triangle-based rasterization methods by operating directly on the parametric domain, allowing for the voxelization of complex implicit or subdivision surfaces without explicit triangulation. This approach is particularly suited for real-time applications where rapid voxelization of non-polygonal geometry is required [2, unofficial].

### [Position Map and Image-Space Transfer](6237-position-map-and-image-space-transfer.md)
This family of techniques converts 3D geometric data into voxel grids by projecting or transferring information into 2D image space (textures) before reconstructing the 3D volume. Unlike direct 3D rasterization, these methods leverage the GPU's fixed-function rasterizer or image load/store operations on 2D textures to determine voxel occupancy, often trading memory bandwidth for computational simplicity. They are distinguished from direct triangle rasterization by operating on pre-computed surface samples or 2D texture slices, ensuring connectivity or enabling real-time performance insensitive to scene complexity.

### Rasterization-based Volume Modeling
The process of building volume data voxel-by-voxel, considered the third layer in the voxel modeling abstraction hierarchy [S5, unofficial]. It solves the problem of assigning density values to voxels based on primitive contributions. It differs from instantiation-based modeling by pulling values into a voxel (similar to raytracing), whereas instantiation-based modeling pushes values into voxels (similar to micropolygon rendering) [S5, unofficial]. The main rasterization loop iterates over voxels in the discrete bounding box, transforms voxel position to world space, samples the primitive, and adds the value to the buffer if positive [S5, unofficial].

### Single-Pass Voxelization
A voxelization approach that utilizes the geometry shader to reduce draw calls by selecting the axis from which each triangle has the largest overall projection for fragment generation [S6, unofficial]. It employs conservative rasterization to generate fragments as long as there is something inside the pixel, not only if its center is covered [S6, unofficial]. It uses an MSAA render target during the voxelization process to solve the issue of incorrectly filling empty cells [S6, unofficial]. Hardware support for conservative rasterization is available on Nvidia Maxwell architecture and later; RTX driver support is available for Turing and Pascal architecture GPUs [S6, unofficial]. A pitfall is that GPUs only generate fragments if a pixel's center is covered by a triangle, leading to small cracks at the edges of triangles, and conservative rasterization may generate too many fragments, incorrectly filling cells in the voxel grid that should be empty [S6, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N_{triangles} \times N_{voxels\_in\_bbox})$ for CPU-based methods, where $N_{voxels\_in\_bbox}$ is the number of voxels in the triangle's bounding box. GPU methods parallelize this, achieving significant speedups (e.g., 50x) for high-resolution grids [S1].
*   **Memory Complexity**: $O(N_{occupied\_voxels})$ for sparse hash-based storage [S3, S4, unofficial]. Dense storage is $O(V_{grid})$, which is inefficient for sparse meshes [S3, unofficial].
*   **Performance**: Output mesh size scales with voxel density. For a torus (441 vertices, 800 faces), voxel size 0.75 results in 61,312 vertices and 91,968 faces, while 0.5 results in 144,968 vertices and 217,452 faces [S2, unofficial].
*   **Strengths**: Simple to implement, hardware-accelerated on GPU, generates explicit surface geometry [S1][S2, S4, unofficial].
*   **Limitations**: May produce holes if not conservative, high vertex count for detailed meshes, surface-only (shell) voxelization [S2, S6, unofficial].
*   **When to use**: Real-time voxel art, surface reconstruction for volume rendering, sparse voxel octrees generation [S1][S2, unofficial].
*   **When not to use**: When deep interior filling is required, or when exact analytic surface representation is needed [S2, S5, unofficial].

## Background
Rasterization-based voxelization originated from the need to convert polygonal models into volumetric data for rendering and simulation [S1][S2, unofficial]. The Point-Cloud-Utils library popularized a Python-based implementation using rasterization to voxels [S1]. The C library `voxelizer` provided a header-only solution for shell voxelization [S2, unofficial]. The `madcad` library introduced the `PositionMap` hasher for sparse voxelization [S3, unofficial]. The technique is widely used in voxel art, real-time ray tracing, and volume rendering pipelines [S1][S5, S6, unofficial].

## Sources
[S1] Voxelizing a Triangle Mesh - Point Cloud Utils, https://fwilliams.info/point-cloud-utils/sections/voxelizing_a_triangle_mesh/, fetched 2026-10-26
[S2] Mesh Voxelization - Geeks3D, https://www.geeks3d.com/20160531/mesh-voxelization/, fetched 2026-10-26 (unofficial)
[S3] In Python, how do I voxelize a 3D mesh - Stack Overflow, https://stackoverflow.com/questions/11851342/in-python-how-do-i-voxelize-a-3d-mesh, fetched 2026-10-26 (unofficial)
[S4] mesh voxelizer library, https://github.com/karimnaaji/voxelizer, fetched 2026-10-26 (unofficial)
[S5] Production Volume Rendering, https://cg.informatik.uni-freiburg.de/intern/seminar/volumeRenderingProductionFundamentals2011.pdf, fetched 2026-10-27 (unofficial)
[S6] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
