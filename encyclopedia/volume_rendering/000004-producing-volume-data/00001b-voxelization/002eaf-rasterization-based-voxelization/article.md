# Rasterization-based Voxelization

Also called: Cube Mesh Generation, Hash-based Surface Voxelization, Sparse Voxelization with PositionMap Hashing, Voxelizing a Triangle Mesh, madcad Voxelization

## Overview
Rasterization-based Voxelization converts a continuous triangle mesh into a discrete volumetric representation by projecting the geometry onto a 3D grid using graphics pipeline primitives or CPU-side scanline algorithms [S1][S3, S4, unofficial]. It solves the problem of generating a voxelized surface (shell) or point cloud from polygonal data without performing complex surface reconstruction [S1][S2, unofficial]. It differs from [Voxel Carving](1213-voxel-carving.md) by using constructive rasterization rather than destructive projection, and from [Analytic and Algebraic Methods](7913-analytic-and-algebraic-methods.md) by relying on discrete grid sampling rather than implicit function evaluation [S1][S5, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangle mesh defined by vertices $V$ and faces $F$ [S1][S4, unofficial].
*   **Parameters:** Voxel size $v_s = (v_{sx}, v_{sy}, v_{sz})$, voxel origin $o$ (typically $[0,0,0]$), and a precision factor $p$ to reduce artifacts [S1][S2, S4, unofficial].
*   **Output:** A sparse set of occupied voxel coordinates, optionally accompanied by a cube mesh geometry or a point cloud with per-voxel attributes [S1][S4, unofficial].

**Data Structures:**
The method typically employs a sparse hash table to store occupied voxels, avoiding the memory overhead of dense 3D arrays [S3, S4, unofficial].
*   **Hash Table:** Maps voxel coordinates to occupancy data. In C implementations, this is a linked-list based hash table where each node contains `position` (voxel center) and `color` [S4, unofficial]. In Python, a standard `set` of coordinate tuples is used [S3, unofficial].
*   **Output Mesh Layout (`vx_mesh_t`):**
    *   `vertices`: Array of size $N_{voxels} \times 8$, storing the 8 corners of each cube.
    *   `normals`: Array of size 6, storing predefined face normals for the cube.
    *   `colors`: Array of size $N_{voxels} \times 8$ (or NULL).
    *   `indices`: Array of size $N_{voxels} \times \text{VOXELIZER\_INDICES\_SIZE}$.
    *   `normalindices`: Array of size $N_{voxels} \times \text{VOXELIZER\_INDICES\_SIZE}$ [S4, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Compute the half-voxel size $h_v = v_s \times 0.5$ [S4, unofficial]. Initialize an empty hash table or set for occupied voxels [S3, S4, unofficial].
2.  **Rasterization Loop:** Iterate through each triangle face $f \in F$ [S3, S4, unofficial].
    *   Determine the bounding box of the triangle in voxel space.
    *   Rasterize the triangle into the grid. This involves identifying all voxel cells $(i, j, k)$ that intersect the triangle [S1][S3, unofficial].
    *   For each intersected voxel, compute its center position $c$ in world space [S4, unofficial].
    *   Insert $c$ into the hash table/set. If the mesh has color attributes, interpolate and store the color at the voxel center [S4, unofficial].
    *   *Note:* The internal rasterizer uses a precision parameter $p$ to handle edge cases and prevent holes [S2, S4, unofficial].
3.  **Geometry Generation (Optional):** If a cube mesh output is required:
    *   Allocate output arrays based on the number of unique voxels $N_{voxels}$ [S4, unofficial].
    *   Define the 8 corner offsets for a cube centered at the origin using $h_v$:
        $$
        \text{offsets} = \{ (\pm h_{vx}, \pm h_{vy}, \pm h_{vz}) \}
        $$
        Specifically: $(-h_{vx}, h_{vy}, h_{vz}), (-h_{vx}, -h_{vy}, h_{vz}), (h_{vx}, -h_{vy}, h_{vz}), (h_{vx}, h_{vy}, h_{vz}), (-h_{vx}, h_{vy}, -h_{vz}), (-h_{vx}, -h_{vy}, -h_{vz}), (h_{vx}, -h_{vy}, -h_{vz}), (h_{vx}, h_{vy}, -h_{vz})$ [S4, unofficial].
    *   For each voxel in the hash table:
        *   Retrieve its center position $P$ and color $C$.
        *   Generate 8 vertices by adding $P$ to each offset.
        *   Assign the predefined face normals to the output mesh [S4, unofficial].
        *   Generate indices to form the 6 faces of the cube.
4.  **Cleanup:** Free the temporary hash table and return the resulting mesh or point cloud [S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rasterization_voxelization(mesh, voxel_size, origin, precision=0.01):
    """
    Voxelize a triangle mesh into a set of occupied voxel centers.
    
    Args:
        mesh: Object with .vertices (N, 3) and .faces (M, 3)
        voxel_size: Tuple (sx, sy, sz)
        origin: Tuple (ox, oy, oz)
        precision: Float for rasterization tolerance
        
    Returns:
        occupied_voxels: Set of (i, j, k) tuples
        voxel_centers: List of (x, y, z) world coordinates
    """
    occupied_voxels = set()
    voxel_centers = []
    
    # Precompute half-voxel size for cube generation if needed
    hvs = [s * 0.5 for s in voxel_size]
    
    for face in mesh.faces:
        # Get triangle vertices in world space
        v0 = mesh.vertices[face[0]]
        v1 = mesh.vertices[face[1]]
        v2 = mesh.vertices[face[2]]
        
        # Determine bounding box in voxel indices
        # Map world coordinates to voxel indices
        # idx = floor((world - origin) / voxel_size)
        min_i = floor(min(v0[0], v1[0], v2[0]) - origin[0]) / voxel_size[0]
        max_i = ceil(max(v0[0], v1[0], v2[0]) - origin[0]) / voxel_size[0]
        min_j = floor(min(v0[1], v1[1], v2[1]) - origin[1]) / voxel_size[1]
        max_j = ceil(max(v0[1], v1[1], v2[1]) - origin[1]) / voxel_size[1]
        min_k = floor(min(v0[2], v1[2], v2[2]) - origin[2]) / voxel_size[2]
        max_k = ceil(max(v0[2], v1[2], v2[2]) - origin[2]) / voxel_size[2]
        
        # Iterate over the bounding box of voxels
        for i in range(int(min_i), int(max_i) + 1):
            for j in range(int(min_j), int(max_j) + 1):
                for k in range(int(min_k), int(max_k) + 1):
                    # Check if the voxel cell intersects the triangle
                    # This requires a robust triangle-box intersection test
                    # [G] Standard implementation uses a conservative rasterization 
                    # or exact geometric overlap test (SAT) to ensure no holes.
                    if triangle_voxel_intersect(v0, v1, v2, i, j, k, voxel_size, origin, precision):
                        occupied_voxels.add((i, j, k))
                        
                        # Compute voxel center in world space
                        center_x = origin[0] + (i + 0.5) * voxel_size[0]
                        center_y = origin[1] + (j + 0.5) * voxel_size[1]
                        center_z = origin[2] + (k + 0.5) * voxel_size[2]
                        voxel_centers.append((center_x, center_y, center_z))
                        
    return occupied_voxels, voxel_centers

def triangle_voxel_intersect(v0, v1, v2, i, j, k, voxel_size, origin, precision):
    """
    [G] Placeholder for the specific intersection logic.
    In C libraries (e.g., voxelizer), this is handled by vx__voxelize which 
    uses a precision factor to clip/intersect.
    In Python (madcad), it uses PositionMap.keysfor() which handles the rasterization.
    """
    # Define voxel bounds in world space
    x_min = origin[0] + i * voxel_size[0]
    x_max = x_min + voxel_size[0]
    y_min = origin[1] + j * voxel_size[1]
    y_max = y_min + voxel_size[1]
    z_min = origin[2] + k * voxel_size[2]
    z_max = z_min + voxel_size[2]
    
    # [G] A robust implementation checks if the triangle intersects the AABB.
    # Simple center-point check is insufficient and causes holes.
    # Conservative rasterization or slab intersection is required.
    return True # Simplified for structure; real impl needs geometric test
```

## Subtopics

### [3D Grid Rasterization](903-3d-grid-rasterization.md)
A CPU-based algorithm that determines which voxels in a fixed-resolution 3D grid are intersected by a triangle mesh, producing a binary occupancy grid. It solves the problem of converting continuous polygonal geometry into a discrete volumetric representation without performing surface reconstruction (such as QEF or dual vertex solving). It differs from full o-voxel implementations by omitting the face QEF, boundary QEF, and dual vertex solve steps, focusing solely on the intersect scanline to find occupied voxels. It differs from GPU-based siblings like [GPU Volume Kernel Voxelization](1496-gpu-volume-kernel-voxelization.md) by relying on CPU-side scanline filling and hash-table deduplication rather than parallel fragment shaders [G].

### [GPU Volume Kernel Voxelization](1496-gpu-volume-kernel-voxelization.md)
GPU Volume Kernel Voxelization is a technique that accelerates the voxelization of triangle meshes by parallelizing the iteration over the 2D XY plane of the voxel grid using a GPU compute shader. It solves the performance bottleneck of CPU-based voxelization, where nested loops over X and Y coordinates are executed sequentially. Unlike [Rasterization-based Voxelization](908-rasterization-based-voxelization.md) and [3D Grid Rasterization](903-3d-grid-rasterization.md) which project geometry into 3D textures, this method retains the explicit triangle-voxel intersection logic of CPU implementations but distributes the workload across GPU threads, achieving significant speedups (e.g., 50x) for high-resolution grids.

### [Implicit Surface Voxelization](2379-implicit-surface-voxelization.md)
Implicit Surface Voxelization converts a mathematical implicit surface definition into a discrete voxel grid by rasterizing 2D rectangles within a Region of Interest (ROI) and evaluating the implicit function in the pixel shader. It differs from triangle-based rasterization methods (e.g., [Rasterization-based Voxelization](908-rasterization-based-voxelization.md)) by operating directly on analytical functions rather than polygonal approximations, allowing for the handling of complex, non-polygonal geometry.

### [GPU Voxelization via Compute Shaders](2549-gpu-voxelization-via-compute-shaders.md)
GPU Voxelization via Compute Shaders converts 3D triangle meshes into a uniform 3D voxel grid by leveraging GPU parallel processing. It solves the problem of generating volumetric data for real-time ray marching or volume rendering with high throughput. Unlike [Rasterization-based Voxelization](908-rasterization-based-voxelization.md), which projects triangles via the fixed-function graphics pipeline, compute shader methods decouple intersection logic from rasterization, allowing custom geometric tests and atomic operations on 3D buffers or textures.

### [GPU Voxelization](2633-gpu-voxelization.md)
GPU Voxelization converts a triangle mesh into a regular voxel grid by leveraging the GPU's hardware rasterization pipeline to map fragments directly to 3D grid cells. It solves the performance bottleneck of CPU-based ray-tracing voxelization by parallelizing the intersection test across the rasterizer. It differs from [3D Grid Rasterization](903-3d-grid-rasterization.md) by using a 3D texture UAV or SSBO as the direct write target rather than a 2D accumulation buffer, and from [GPU Volume Kernel Voxelization](1496-gpu-volume-kernel-voxelization.md) by relying on standard triangle rasterization rather than volume kernels.

### [CPU Mesh Voxelization](3137-cpu-mesh-voxelization.md)
CPU Mesh Voxelization converts a triangle mesh into a 3D grid of voxels by iterating over the bounding box of each triangle and testing each voxel center against the triangle's plane and edge constraints. It differs from its parent [Rasterization-based Voxelization](908-rasterization-based-voxelization.md) by outputting a binary occupancy grid or bit-packed table rather than generating a cube mesh per occupied voxel. It differs from [3D Grid Rasterization](903-3d-grid-rasterization.md) by using exact geometric overlap tests (SAT or plane/projection) rather than simple grid cell intersection, and from [Voxelization via Triangle Scattering](3779-voxelization-via-triangle-scattering.md) by iterating cells against triangles rather than scattering triangles into cells.

### [Voxel Rasterization](3204-voxel-rasterization.md)
Voxel Rasterization converts a triangle mesh into a voxelized representation by rendering the geometry into a 3D voxel buffer, supporting both simple occupancy and normal/color mapping. It differs from its sibling [3D Grid Rasterization](903-3d-grid-rasterization.md) by explicitly iterating over triangle surfaces to calculate interpolated points and normals rather than relying on a pure grid traversal, and from [GPU Volume Kernel Voxelization](1496-gpu-volume-kernel-voxelization.md) by utilizing a Direct3D 11-based rendering pipeline or CUDA kernels for the rasterization step.

### [Conservative Voxelization](3247-conservative-voxelization.md)
Conservative voxelization is a hardware-accelerated algorithm for polygonal models that guarantees conservative correctness: every voxel intersecting the input model is correctly recognized, solving the problem where previous approaches miss intersecting voxels. It differs from standard rasterization-based voxelization [Rasterization-based Voxelization] by computing the exact depth range of the intersection between the triangle and the voxel column, rather than using a single depth value (e.g., the fragment center) which leads to missed voxels.

### [Conservative Rasterization via Triangle Expansion](3531-conservative-rasterization-via-triangle-expansion.md)
This method eliminates voxelization holes caused by standard rasterization, which only covers pixels whose centers lie inside a triangle, by expanding the triangle in clip space and culling the expanded area in the fragment shader. It differs from sibling methods like [Conservative Voxelization](3247-conservative-voxelization.md) by implementing the expansion logic manually in the geometry shader using homogeneous plane mathematics, rather than relying on hardware-specific conservative rasterization extensions or post-processing hole-filling.

### [Multi-view projection](5937-multi-view-projection.md)
Multi-view projection voxelization converts a triangle mesh into a voxel grid by rasterizing the geometry from multiple camera viewpoints, typically aligned with the principal axes of the scene's bounding box. This approach solves the coverage gaps and geometric distortions inherent in single-view projection methods, which fail when surface normals are perpendicular to the view direction. It differs from single-view methods by ensuring comprehensive coverage through redundancy, and from compute-shader-based methods by leveraging standard graphics pipeline rasterization.

### [Parametric Surface Voxelization](6032-parametric-surface-voxelization.md)
Parametric Surface Voxelization converts a parametric surface (such as a subdivision surface or implicit field) into a voxel grid by rasterizing points in the 2D parameter space rather than the 3D object space. It differs from triangle-based rasterization methods by operating directly on the parametric domain, allowing for the voxelization of complex implicit or subdivision surfaces without explicit triangulation. This approach is particularly suited for real-time applications where rapid voxelization of non-polygonal geometry is required.

### [Position Map and Image-Space Transfer](6237-position-map-and-image-space-transfer.md)
This family of techniques converts 3D geometric data into voxel grids by projecting or transferring information into 2D image space (textures) before reconstructing the 3D volume [G]. Unlike direct 3D rasterization, these methods leverage the GPU's fixed-function rasterizer or image load/store operations on 2D textures to determine voxel occupancy [G]. They are distinguished from direct triangle rasterization by operating on pre-computed surface samples or 2D texture slices, ensuring connectivity or enabling real-time performance insensitive to scene complexity [G].

### Rasterization-based Volume Modeling
The process of building volume data voxel-by-voxel, considered the third layer in the voxel modeling abstraction hierarchy [S5, unofficial]. It involves instancing the primitive representation, bounding it to find the region of voxels it overlaps, and sampling its density function at each voxel in the overlapping region [S5, unofficial]. The main rasterization loop iterates over voxels in the discrete bounding box, transforms voxel position to world space, samples the primitive, and adds the value to the buffer if positive [S5, unofficial]. Rasterization-based modeling pulls values into a voxel (similar to raytracing), whereas instantiation-based modeling pushes values into voxels (similar to micropolygon rendering) [S5, unofficial].

### Single-Pass Voxelization
A voxelization approach that utilizes the geometry shader to reduce draw calls by selecting the axis from which each triangle has the largest overall projection for fragment generation [S6, unofficial]. It employs conservative rasterization to generate fragments as long as there is something inside the pixel, not only if its center is covered [S6, unofficial]. It uses an MSAA render target during the voxelization process to solve the issue of incorrectly filling empty cells [S6, unofficial]. Hardware support for conservative rasterization is available on Nvidia Maxwell architecture and later; RTX driver support is available for Turing and Pascal architecture GPUs [S6, unofficial]. A pitfall is that GPUs only generate fragments if a pixel's center is covered by a triangle, leading to small cracks at the edges of triangles, and conservative rasterization can generate too many fragments, incorrectly filling cells in the voxel grid that should be empty [S6, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** Complexity is proportional to the number of triangles $N_f$ multiplied by the average number of voxels intersected per triangle. For high-resolution grids, this can be expensive on CPU [S1][S3, unofficial]. GPU implementations offer significant speedups (e.g., 50x) for high-resolution grids [S1].
*   **Memory:** Using a sparse hash table (set) makes it memory-efficient for sparse meshes compared to dense 3D boolean arrays [S3, S4, unofficial]. However, the output cube mesh scales with voxel density; for a torus mesh (441 vertices, 800 faces), a voxel size of 0.75 results in 61,312 vertices and 91,968 faces, while 0.5 results in 144,968 vertices and 217,452 faces [S2, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Simple to implement; generates explicit cube meshes or point clouds; handles sparse data efficiently via hashing [S1][S3, S4, unofficial].
*   **Limitations:** Output mesh size grows rapidly with resolution, impacting rendering performance [S2, unofficial]. Standard rasterization may miss voxels (holes) if not using conservative rasterization or precise geometric tests [S1][S6, unofficial].

**When to Use:**
*   Use when an explicit surface mesh (cube mesh) or point cloud is required from a triangle mesh.
*   Use when memory efficiency for sparse volumes is critical (hash-based approach).
*   Avoid when deep (solid) voxelization is required, as this method typically performs shell voxelization [S2, unofficial].

## Background
Originating from techniques to rasterize triangle meshes to voxels for point cloud processing and voxel art, this method is implemented in libraries such as Point-Cloud-Utils [S1] and the header-only C library `voxelizer` [S4, unofficial]. It is also used in Python via `madcad`'s `PositionMap` [S3, unofficial]. The approach is foundational for converting polygonal assets into volumetric formats for rendering or simulation [S1][S2, unofficial].

## Sources
[S1] Voxelizing a Triangle Mesh - Point Cloud Utils, https://fwilliams.info/point-cloud-utils/sections/voxelizing_a_triangle_mesh/, fetched 2026-10-26
[S2] Mesh Voxelization - Geeks3D, https://www.geeks3d.com/20160531/mesh-voxelization/, fetched 2026-10-26 (unofficial)
[S3] In Python, how do I voxelize a 3D mesh - Stack Overflow, https://stackoverflow.com/questions/11851342/in-python-how-do-i-voxelize-a-3d-mesh, fetched 2026-10-26 (unofficial)
[S4] mesh voxelizer library, https://github.com/karimnaaji/voxelizer, fetched 2026-10-26 (unofficial)
[S5] Production Volume Rendering, https://cg.informatik.uni-freiburg.de/intern/seminar/volumeRenderingProductionFundamentals2011.pdf, fetched 2026-10-27 (unofficial)
[S6] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
