# Rasterization-based Voxelization

Also called: Cube Mesh Generation, Hash-based Surface Voxelization, Sparse Voxelization with PositionMap Hashing, Voxelizing a Triangle Mesh, madcad Voxelization

## Overview
Rasterization-based Voxelization converts a continuous triangle mesh into a discrete volumetric representation by projecting the geometry onto a 3D grid using graphics pipeline primitives or CPU-side scanline algorithms [S1][S4, unofficial]. It solves the problem of generating a voxelized surface (shell) or point cloud from polygonal data without performing implicit surface reconstruction [S1][S2, unofficial]. It differs from [Voxel Carving](1213-voxel-carving.md) by using constructive geometry intersection rather than destructive view-projection, and from [Implicit Surface Voxelization](2379-implicit-surface-voxelization.md) by operating directly on explicit triangle primitives rather than signed distance fields [S1][S2, unofficial].

## How it works
The method maps continuous world-space coordinates to discrete integer grid indices $(i, j, k)$ [G]. The core mechanism involves coordinate mapping, occupancy determination via rasterization, and attribute assignment [G].

### Inputs and Outputs
**Inputs:**
*   Input mesh vertices $V$ and faces $F$ (triangles) [S1][S4, unofficial].
*   Voxel size $v_s = (v_{sx}, v_{sy}, v_{sz})$ [S4, unofficial].
*   Voxel origin $o = (o_x, o_y, o_z)$, the bottom-left-back corner of the $(0,0,0)$ voxel [S1].
*   Precision factor $p$ to reduce artifacts [S2, S4, unofficial].
*   Gap fraction $g$ for cube generation [S1].

**Outputs:**
*   Occupied voxel coordinates (integer grid indices) [S1].
*   Cube mesh: vertices and indices representing the voxelized surface [S1][S4, unofficial].
*   Point cloud: voxel centers with optional colors [S4, unofficial].

### Data Structures
**Hash Table (Sparse Storage):**
Used to store occupied voxels to avoid dense 3D arrays [S3, S4, unofficial].
*   **Key:** Voxel center position or grid index.
*   **Value:** `voxel_data_t` containing:
    *   `position`: World-space center of the voxel.
    *   `color`: Optional per-voxel color attribute.
*   **Implementation:** A hash table with buckets and linked lists for collision resolution [S4, unofficial].

**Output Mesh Structure (`vx_mesh_t`):**
*   `vertices`: Array of size $N_{voxels} \times 8$, storing 8 corners per cube [S4, unofficial].
*   `normals`: Array of size 6, storing predefined face normals (18 floats) [S4, unofficial].
*   `colors`: Array of size $N_{voxels} \times 8$, or NULL [S4, unofficial].
*   `indices`: Array of size $N_{voxels} \times \text{VOXELIZER_INDICES\_SIZE}$ [S4, unofficial].
*   `normalindices`: Array of size $N_{voxels} \times \text{VOXELIZER\_INDICES\_SIZE}$ [S4, unofficial].

### Steps
1.  **Initialization:**
    *   Compute half-voxel size $h_v = v_s \times 0.5$ [S4, unofficial].
    *   Initialize an empty hash table `table` [S4, unofficial].
    *   Initialize voxel counter $N_{voxels} = 0$ [S4, unofficial].

2.  **Rasterization (Occupancy Determination):**
    *   Iterate through each triangle face $f \in F$ [S3, S4, unofficial].
    *   For each face, determine the set of voxel grid cells intersected by the triangle [S1][S3, unofficial].
        *   *CPU/Hash Method:* Use a `PositionMap` hasher or scanline algorithm to find voxel keys corresponding to the triangle's projection [S3, S4, unofficial].
        *   *GPU Method:* Rasterize the triangle into a 3D texture or UAV, writing to voxel coordinates [S6, unofficial].
    *   For each intersected voxel:
        *   Calculate the voxel center $c$ in world space.
        *   Compute hash key for $c$ [S4, unofficial].
        *   Insert $(c, \text{color})$ into `table` if not present [S4, unofficial].
        *   Increment $N_{voxels}$ if a new voxel was inserted [S4, unofficial].

3.  **Geometry Generation (Cube Mesh):**
    *   Allocate output mesh arrays based on $N_{voxels}$ [S4, unofficial].
    *   Copy predefined face normals to `outmesh->normals` [S4, unofficial].
    *   Define 8 cube corner offsets relative to center:
        $$
        \begin{aligned}
        &(-h_{vx}, h_{vy}, h_{vz}), (-h_{vx}, -h_{vy}, h_{vz}), (h_{vx}, -h_{vy}, h_{vz}), (h_{vx}, h_{vy}, h_{vz}), \\
        &(-h_{vx}, h_{vy}, -h_{vz}), (-h_{vx}, -h_{vy}, -h_{vz}), (h_{vx}, -h_{vy}, -h_{vz}), (h_{vx}, h_{vy}, -h_{vz})
        \end{aligned}
        $$
        [S4, unofficial].
    *   Iterate through `table` buckets [S4, unofficial]:
        *   For each node, extract `voxeldata->position` and `voxeldata->color`.
        *   Call `vx__add_voxel` to populate `outmesh->vertices`, `indices`, and `colors` using the offsets and extracted data [S4, unofficial].

4.  **Cleanup:**
    *   Free the hash table [S4, unofficial].
    *   Return the output mesh or point cloud [S4, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rasterization_voxelization(mesh, voxel_size, origin, precision=0.01):
    """
    Voxelize a triangle mesh into a cube mesh using rasterization principles.
    mesh: list of triangles, each triangle is a list of 3 vertices (vec3)
    voxel_size: float or vec3, size of each voxel
    origin: vec3, bottom-left-back corner of the (0,0,0) voxel
    precision: float, tolerance for intersection
    """
    # 1. Initialization
    hvs = voxel_size * 0.5
    hash_table = {}  # Key: voxel_center, Value: color
    voxel_centers = []
    
    # 2. Rasterization
    for triangle in mesh.faces:
        # Determine intersected voxels
        # This step varies by implementation (CPU scanline vs GPU rasterization)
        # Here we assume a helper function that returns voxel centers
        intersected_centers = rasterize_triangle_to_voxels(triangle, voxel_size, origin, precision)
        
        for center in intersected_centers:
            if center not in hash_table:
                hash_table[center] = triangle.get_color() # Interpolate color if needed
                voxel_centers.append(center)
                
    # 3. Geometry Generation
    n_voxels = len(voxel_centers)
    cube_vertices = []
    cube_indices = []
    cube_colors = []
    
    # Predefined cube corner offsets relative to center
    offsets = [
        (-hvs.x,  hvs.y,  hvs.z), (-hvs.x, -hvs.y,  hvs.z),
         hvs.x, -hvs.y,  hvs.z),  (hvs.x,  hvs.y,  hvs.z),
        (-hvs.x,  hvs.y, -hvs.z), (-hvs.x, -hvs.y, -hvs.z),
         hvs.x, -hvs.y, -hvs.z),  (hvs.x,  hvs.y, -hvs.z)
    ]
    
    # Standard cube face indices (0-7)
    # Note: VOXELIZER_INDICES_SIZE is typically 36 for 12 triangles (36 indices)
    # Or 24 for 6 quads (24 indices) depending on output format. 
    # Using 36 indices for 12 triangles (2 per face) as per common cube mesh gen.
    face_indices = [
        0, 1, 2, 0, 2, 3,  # Front
        4, 6, 5, 4, 7, 6,  # Back
        4, 5, 1, 4, 1, 0,  # Left
        3, 2, 6, 3, 6, 7,  # Right
        3, 7, 4, 3, 4, 0,  # Top
        1, 5, 6, 1, 6, 2   # Bottom
    ]
    
    for center in voxel_centers:
        base_idx = len(cube_vertices)
        color = hash_table[center]
        
        for offset in offsets:
            cube_vertices.append(center + offset)
            cube_colors.append(color)
            
        for idx in face_indices:
            cube_indices.append(base_idx + idx)
            
    return cube_vertices, cube_indices, cube_colors

def rasterize_triangle_to_voxels(triangle, voxel_size, origin, precision):
    """
    [G] Placeholder for specific rasterization logic.
    In CPU implementations (e.g., madcad), this uses a PositionMap hasher.
    In GPU implementations, this is handled by the graphics pipeline.
    Returns a list of voxel centers intersected by the triangle.
    """
    # Implementation detail omitted: depends on specific rasterizer backend
    # Must return all voxel centers whose AABB intersects the triangle
    pass
```

### Parameters
*   **voxel_size**: Physical dimension of a voxel. Typical values: $0.75, 0.5, 0.2, 0.1$ units [S2, unofficial]. In Point-Cloud-Utils, often $1.0/256.0$ [S1].
*   **vox_origin**: Location of the $(0,0,0)$ voxel corner. Typically $[0, 0, 0]$ [S1].
*   **precision**: Factor to reduce "holes" artifacts. Typical value: $0.01$ [S2, S4, unofficial].
*   **gap_fraction**: Spacing between cubes in the generated geometry. Typical value: $0.02$ [S1].

### Edge Cases and Pitfalls
*   **Holes/Artifacts:** Standard rasterization may miss voxels if triangle edges fall between voxel centers. A precision factor or conservative rasterization is required to mitigate this [S2, S6, unofficial].
*   **Performance:** The output mesh size scales with voxel density. A torus mesh (441 vertices, 800 faces) with voxel size $0.75$ results in 61,312 vertices and 91,968 faces [S2, unofficial].
*   **Sparse vs Dense:** Using a hash table (sparse) is more memory-efficient for sparse meshes than a 3D boolean array [S3, S4, unofficial].

## Subtopics

### [3D Grid Rasterization](903-3d-grid-rasterization.md)
A CPU-based algorithm that determines which voxels in a fixed-resolution 3D grid are intersected by a triangle mesh, producing a binary occupancy grid. It solves the problem of converting continuous polygonal geometry into a discrete volumetric representation without performing surface reconstruction (such as QEF or dual vertex solving). It differs from full o-voxel implementations by omitting the face QEF, boundary QEF, and dual vertex solve steps, focusing solely on the intersect scanline to find occupied voxels. It differs from GPU-based siblings like [GPU Volume Kernel Voxelization](1496-gpu-volume-kernel-voxelization.md) by relying on CPU-side scanline filling and hash-table deduplication rather than parallel fragment shaders [G].

### [GPU Volume Kernel Voxelization](1496-gpu-volume-kernel-voxelization.md)
GPU Volume Kernel Voxelization is a technique that accelerates the voxelization of triangle meshes by parallelizing the iteration over the 2D XY plane of the voxel grid using a GPU compute shader. It solves the performance bottleneck of CPU-based voxelization, where nested loops over X and Y coordinates are executed sequentially. Unlike [Rasterization-based Voxelization](908-rasterization-based-voxelization.md) and [3D Grid Rasterization](903-3d-grid-rasterization.md) which project geometry into 3D textures, this method retains the explicit triangle-voxel intersection logic of CPU implementations but distributes the workload across GPU threads, achieving significant speedups (e.g., 50x) for high-resolution grids.

### [GPU Voxelization via Compute Shaders](2549-gpu-voxelization-via-compute-shaders.md)
GPU Voxelization via Compute Shaders converts 3D triangle meshes into a uniform 3D voxel grid by leveraging GPU parallel processing. It solves the problem of generating volumetric data for real-time ray marching or volume rendering with high throughput. Unlike [Rasterization-based Voxelization](908-rasterization-based-voxelization.md), which projects triangles via the fixed-function graphics pipeline, compute shader methods decouple intersection logic from rasterization, allowing custom geometric tests and atomic operations on 3D buffers or textures.

### [GPU Voxelization](2633-gpu-voxelization.md)
GPU Voxelization converts a triangle mesh into a regular voxel grid by leveraging the GPU's hardware rasterization pipeline to map fragments directly to 3D grid cells. It solves the performance bottleneck of CPU-based ray-tracing voxelization by parallelizing the intersection test across the rasterizer. It differs from [3D Grid Rasterization](903-3d-grid-rasterization.md) by using a 3D texture UAV or SSBO as the direct write target rather than a 2D accumulation buffer, and from [GPU Volume Kernel Voxelization](1496-gpu-volume-kernel-voxelization.md) by relying on standard triangle rasterization rather than volume kernels.

### [CPU Mesh Voxelization](3137-cpu-mesh-voxelization.md)
CPU Mesh Voxelization converts a triangle mesh into a 3D grid of voxels by iterating over the bounding box of each triangle and testing each voxel center against the triangle's plane and edge constraints. It differs from its parent [Rasterization-based Voxelization](908-rasterization-based-voxelization.md) by outputting a binary occupancy grid or bit-packed table rather than generating a cube mesh per occupied voxel. It differs from [3D Grid Rasterization](903-3d-grid-rasterization.md) by using exact geometric overlap tests (SAT or plane/projection) rather than simple grid cell intersection, and from [Voxelization via Triangle Scattering](3779-voxelization-via-triangle-scattering.md) by iterating cells against triangles rather than scattering triangles into cells.

### [Voxel Rasterization](3204-voxel-rasterization.md)
Voxel Rasterization converts a triangle mesh into a voxelized representation by rendering the geometry into a 3D voxel buffer, supporting both simple occupancy and normal/color mapping. It differs from its sibling [3D Grid Rasterization](903-3d-grid-rasterization.md) by explicitly iterating over triangle surfaces to calculate interpolated points and normals rather than relying on a pure grid traversal, and from [GPU Volume Kernel Voxelization](1496-gpu-volume-kernel-voxelization.md) by utilizing a Direct3D 11-based rendering pipeline or CUDA kernels for the rasterization step.

### [Conservative Voxelization](3247-conservative-voxelization.md)
Conservative voxelization is a hardware-accelerated algorithm for polygonal models that guarantees conservative correctness: every voxel intersecting the input model is correctly recognized. It differs from standard rasterization-based voxelization [Rasterization-based Voxelization] by computing the exact depth range of the intersection between the triangle and the voxel column, rather than using a single depth value which leads to missed voxels.

### [Conservative Rasterization via Triangle Expansion](3531-conservative-rasterization-via-triangle-expansion.md)
This method eliminates voxelization holes caused by standard rasterization, which only covers pixels whose centers lie inside a triangle, by expanding the triangle in clip space and culling the expanded area in the fragment shader. It differs from sibling methods like [Conservative Voxelization](3247-conservative-voxelization.md) by implementing the expansion logic manually in the geometry shader using homogeneous plane mathematics, rather than relying on hardware-specific conservative rasterization extensions or post-processing hole-filling.

### [Multi-view projection](5937-multi-view-projection.md)
Multi-view projection voxelization converts a triangle mesh into a voxel grid by rasterizing the geometry from multiple camera viewpoints, typically aligned with the principal axes of the scene's bounding box. This approach solves the coverage gaps and geometric distortions inherent in single-view projection methods, which fail when surface normals are perpendicular to the view direction. It differs from single-view methods by ensuring comprehensive coverage through redundancy, and from compute-shader-based methods by leveraging standard graphics pipeline rasterization.

### [Position Map and Image-Space Transfer](6237-position-map-and-image-space-transfer.md)
This family of techniques converts 3D geometric data into voxel grids by projecting or transferring information into 2D image space (textures) before reconstructing the 3D volume [G]. Unlike direct 3D rasterization, these methods leverage the GPU's fixed-function rasterizer or image load/store operations on 2D textures to determine voxel occupancy [G]. They are distinguished from direct triangle rasterization by operating on pre-computed surface samples or 2D texture slices, ensuring connectivity or enabling real-time performance insensitive to scene complexity [G].

#### Rasterization-based Volume Modeling
The process of building volume data voxel-by-voxel, considered the third layer in the voxel modeling abstraction hierarchy [S5, unofficial]. It solves the problem of assigning density values to voxels based on primitive contributions. It differs from instantiation-based modeling by pulling values into a voxel (similar to raytracing), whereas instantiation-based modeling pushes values into voxels (similar to micropolygon rendering) [S5, unofficial]. The main rasterization loop iterates over voxels in the discrete bounding box, transforms voxel position to world space, samples the primitive, and adds the value to the buffer if positive [S5, unofficial].

#### Single-Pass Voxelization
A voxelization approach that utilizes the geometry shader to reduce draw calls by selecting the axis from which each triangle has the largest overall projection for fragment generation [S6, unofficial]. It solves the problem of multiple draw calls required for multi-axis voxelization. It employs conservative rasterization to generate fragments as long as there is something inside the pixel, not only if its center is covered [S6, unofficial]. Hardware support for conservative rasterization is available on Nvidia Maxwell architecture and later; RTX driver support is available for Turing and Pascal architecture GPUs [S6, unofficial]. A pitfall is that GPUs only generate fragments if a pixel's center is covered by a triangle, leading to small cracks at the edges of triangles, and conservative rasterization may incorrectly fill empty cells [S6, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of triangles and $M$ is the average number of voxels intersected per triangle [S1][S4, unofficial].
*   **Memory Complexity:** $O(V)$, where $V$ is the number of occupied voxels, stored in a hash table [S3, S4, unofficial].
*   **Performance:** GPU-based methods achieve significant speedups (e.g., 50x) over CPU methods for high-resolution grids [S1].
*   **Strengths:** Efficient for sparse meshes; generates explicit cube meshes suitable for rendering; supports attribute mapping (color/normal) [S1][S4, unofficial].
*   **Limitations:** Output mesh size can be very large compared to input (e.g., 61k vertices for a simple torus) [S2, unofficial]; requires conservative rasterization or precision tuning to avoid holes [S2, S6, unofficial].
*   **When to use:** When an explicit voxelized mesh representation is needed for rendering or further processing, and the mesh is sparse or high-resolution [S1][S4, unofficial].
*   **When not to use:** When a dense occupancy grid is sufficient and memory is constrained (use sparse hash only if necessary); when deep voxelization (interior) is required (this method is shell-only) [S2, unofficial].

## Background
Rasterization-based voxelization originated from the need to convert polygonal models into voxel data for volume rendering and voxel art [S1][S2, unofficial]. The Point-Cloud-Utils library popularized a CPU-based implementation using rasterization principles [S1]. The C library `voxelizer` by Karim Naaji provided a header-only solution for shell voxelization [S2, unofficial]. The madcad library introduced the `PositionMap` hasher for efficient sparse voxelization [S3, unofficial]. GPU-based variants leveraged the graphics pipeline's hardware rasterization for acceleration [S6, unofficial].

## Sources
[S1] Voxelizing a Triangle Mesh - Point Cloud Utils, https://fwilliams.info/point-cloud-utils/sections/voxelizing_a_triangle_mesh/, fetched 2026-10-26
[S2] Mesh Voxelization - Geeks3D, https://www.geeks3d.com/20160531/mesh-voxelization/, fetched 2026-10-26 (unofficial)
[S3] In Python, how do I voxelize a 3D mesh - Stack Overflow, https://stackoverflow.com/questions/11851342/in-python-how-do-i-voxelize-a-3d-mesh, fetched 2026-10-26 (unofficial)
[S4] mesh voxelizer library, https://github.com/karimnaaji/voxelizer, fetched 2026-10-26 (unofficial)
[S5] Production Volume Rendering, https://cg.informatik.uni-freiburg.de/intern/seminar/volumeRenderingProductionFundamentals2011.pdf, fetched 2026-10-27 (unofficial)
[S6] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
