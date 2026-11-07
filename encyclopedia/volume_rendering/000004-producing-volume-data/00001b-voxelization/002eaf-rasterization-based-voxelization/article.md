# Rasterization-based Voxelization

Also called: Cube Mesh Generation, Hash-based Surface Voxelization, Sparse Voxelization with PositionMap Hashing, Voxelizing a Triangle Mesh, madcad Voxelization

## Overview
Rasterization-based Voxelization converts a continuous triangle mesh into a discrete volumetric representation by projecting the geometry onto a 3D grid using rasterization principles [S1][S3, S4, unofficial]. It solves the problem of generating a voxelized surface (shell) or point cloud from polygonal data without performing complex surface reconstruction [S1][S2, unofficial]. Unlike [Voxel Carving](1213-voxel-carving.md) which uses volume intersection, or [Implicit Surface Voxelization](2379-implicit-surface-voxelization.md) which samples distance fields, this method explicitly rasterizes triangles to determine occupied grid cells, often leveraging hash tables for sparse storage [S3, S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangle mesh defined by vertices $V$ and faces $F$ [S1][S4, unofficial].
*   **Parameters:** Voxel size $v_s = (v_x, v_y, v_z)$ [S4, unofficial], voxel origin $o$ (typically $[0,0,0]$) [S1], and a precision factor $p$ to reduce artifacts [S2, S4, unofficial].
*   **Output:** A set of occupied voxel coordinates (integer indices $ijk$) [S1], a sparse hash table of voxel data [S4, unofficial], or a generated cube mesh/point cloud representing the voxelized surface [S1][S4, unofficial].

**Data Structures:**
1.  **Hash Table:** Used to store occupied voxels to avoid dense 3D arrays. Keys are voxel coordinates; values contain position and optional color/normal data [S3, S4, unofficial].
2.  **Voxel Data Node:** Contains `position` (world space center) and `color` [S4, unofficial].
3.  **Output Mesh Structure (`vx_mesh_t`):**
    *   `vertices`: Array of size $N_{voxels} \times 8$ [S4, unofficial].
    *   `normals`: Array of size 6 (predefined cube normals) [S4, unofficial].
    *   `colors`: Array of size $N_{voxels} \times 8$ (optional) [S4, unofficial].
    *   `indices`: Array of size $N_{voxels} \times \text{VOXELIZER_INDICES_SIZE}$ [S4, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    *   Load mesh vertices $V$ and faces $F$ [S1].
    *   Compute half-voxel size $h_v = v_s \times 0.5$ [S4, unofficial].
    *   Initialize an empty hash table $H$ and voxel counter $N_{voxels} = 0$ [S4, unofficial].

2.  **Rasterization (Triangle Iteration):**
    *   For each triangle face $f \in F$:
        *   Determine the bounding box of the triangle in voxel space.
        *   Iterate through all integer voxel coordinates $(i, j, k)$ within this bounding box [S5, unofficial].
        *   For each voxel, check if it intersects the triangle. This is often done by checking if the voxel center lies within the triangle's projection or by conservative rasterization logic [S1][S6, unofficial].
        *   If intersected, compute the voxel center in world space: $C = o + (i + 0.5, j + 0.5, k + 0.5) \times v_s$ [G].
        *   Insert $(C, \text{color})$ into $H$ using a hash function $h(C)$ [S4, unofficial]. If the voxel was not already present, increment $N_{voxels}$ [S4, unofficial].

3.  **Geometry Generation (Optional):**
    *   Allocate output arrays for $N_{voxels}$ cubes [S4, unofficial].
    *   Define the 8 corner offsets for a cube centered at origin:
        $$
        \text{offsets} = \begin{bmatrix}
        -h_x, h_y, h_z & -h_x, -h_y, h_z & h_x, -h_y, h_z & h_x, h_y, h_z \\
        -h_x, h_y, -h_z & -h_x, -h_y, -h_z & h_x, -h_y, -h_z & h_x, h_y, -h_z
        \end{bmatrix}
        $$
    *   For each unique voxel in $H$:
        *   Retrieve position $P$ and color $C_{col}$ [S4, unofficial].
        *   Generate 8 vertices by adding offsets to $P$ [S4, unofficial].
        *   Assign predefined face normals [S4, unofficial].
        *   Store vertices and indices in the output mesh [S4, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rasterization_voxelize(mesh_vertices, mesh_faces, voxel_size, origin, precision):
    """
    Rasterizes a triangle mesh into a sparse set of voxel coordinates.
    
    Args:
        mesh_vertices: List of 3D coordinates (N, 3)
        mesh_faces: List of vertex indices (M, 3)
        voxel_size: Float or tuple (vx, vy, vz)
        origin: Tuple (ox, oy, oz)
        precision: Float tolerance for intersection
        
    Returns:
        set: A set of integer voxel coordinates (i, j, k)
    """
    voxels = set()
    
    # Precompute half voxel size for center calculation if needed
    # Note: The core rasterization logic depends on the specific implementation
    # (e.g., madcad's PositionMap or Point-Cloud-Utils).
    # Here we outline the generic hash-based approach described in [S3, S4, unofficial].
    
    for face in mesh_faces:
        # Get triangle vertices
        v0 = mesh_vertices[face[0]]
        v1 = mesh_vertices[face[1]]
        v2 = mesh_vertices[face[2]]
        
        # Compute bounding box in voxel space
        # Map world coords to voxel indices
        # idx = floor((coord - origin) / voxel_size)
        min_i = min(floor((v0.x, v1.x, v2.x) - origin) / voxel_size)
        max_i = max(floor((v0.x, v1.x, v2.x) - origin) / voxel_size)
        # Repeat for j and k
        
        # Iterate over voxels in bounding box
        for i in range(min_i, max_i + 1):
            for j in range(min_j, max_j + 1):
                for k in range(min_k, max_k + 1):
                    # Check intersection
                    # This step varies by implementation.
                    # Simple approach: Check if voxel center is inside triangle
                    # or use conservative rasterization logic [S6, unofficial].
                    if intersects_triangle(v0, v1, v2, i, j, k, voxel_size, origin, precision):
                        voxels.add((i, j, k))
                        
    return voxels

def intersects_triangle(v0, v1, v2, i, j, k, voxel_size, origin, precision):
    """
    Placeholder for specific intersection test.
    In madcad [S3, unofficial], this uses PositionMap.keysfor().
    In Point-Cloud-Utils [S1], this is handled by pcu.voxelize_triangle_mesh.
    A common geometric test involves checking if the voxel box intersects the triangle plane.
    """
    # Implementation detail: [G]
    # Calculate voxel center
    cx = origin.x + (i + 0.5) * voxel_size.x
    cy = origin.y + (j + 0.5) * voxel_size.y
    cz = origin.z + (k + 0.5) * voxel_size.z
    center = (cx, cy, cz)
    
    # Check if center is inside triangle (barycentric coords)
    # Or use a more robust box-triangle intersection test
    return point_in_triangle(center, v0, v1, v2)

def point_in_triangle(p, v0, v1, v2):
    """
    Simple barycentric check. Not robust for all cases.
    """
    # Compute vectors
    v0v1 = v1 - v0
    v0v2 = v2 - v0
    v0p = p - v0
    
    # Compute dot products
    dot00 = dot(v0v2, v0v2)
    dot01 = dot(v0v2, v0v1)
    dot02 = dot(v0v2, v0p)
    dot11 = dot(v0v1, v0v1)
    dot12 = dot(v0v1, v0p)
    
    # Compute barycentric coordinates
    inv_denom = 1 / (dot00 * dot11 - dot01 * dot01)
    u = (dot11 * dot02 - dot01 * dot12) * inv_denom
    v = (dot00 * dot12 - dot01 * dot02) * inv_denom
    
    # Check if point is in triangle
    return (u >= 0) and (v >= 0) and (u + v <= 1)
```

**Edge Cases and Pitfalls:**
*   **Holes/Artifacts:** Standard rasterization may miss voxels if the triangle is thin or aligned with the grid. A precision parameter (e.g., 0.01f) is used to expand the intersection test slightly [S2, S4, unofficial].
*   **Performance:** The output mesh size scales with voxel density. A torus with 800 faces can result in >140,000 vertices at small voxel sizes [S2, unofficial].
*   **Sparse Storage:** Using a hash set instead of a 3D array is critical for memory efficiency on sparse meshes [S3, S4, unofficial].

## Subtopics

### [3D Grid Rasterization](903-3d-grid-rasterization.md)
A CPU-based algorithm that determines which voxels in a fixed-resolution 3D grid are intersected by a triangle mesh, producing a binary occupancy grid or a grid with interpolated colors. It solves the problem of converting continuous polygonal geometry into a discrete volumetric representation without performing surface reconstruction (such as QEF or dual vertex solving). It differs from full o-voxel implementations by omitting the face QEF, boundary QEF, and dual vertex solve steps, focusing solely on the intersect scanline to find occupied voxels. It differs from GPU-based siblings like [GPU Volume Kernel Voxelization](1496-gpu-volume-kernel-voxelization.md) by relying on CPU-side scanline filling and hash-table deduplication rather than parallel fragment shaders.

### [GPU Volume Kernel Voxelization](1496-gpu-volume-kernel-voxelization.md)
GPU Volume Kernel Voxelization is a technique that accelerates the voxelization of triangle meshes by parallelizing the iteration over the 2D XY plane of the voxel grid using a GPU compute shader. It solves the performance bottleneck of CPU-based voxelization, where nested loops over X and Y coordinates are executed sequentially. Unlike rasterization-based alternatives that project geometry into 3D textures, this method retains the explicit triangle-voxel intersection logic of CPU implementations but distributes the workload across GPU threads, achieving significant speedups (e.g., 50x) for high-resolution grids.

### [GPU Voxelization via Compute Shaders](2549-gpu-voxelization-via-compute-shaders.md)
GPU Voxelization via Compute Shaders converts 3D triangle meshes into a uniform 3D voxel grid by leveraging GPU parallel processing. It solves the problem of generating volumetric data for real-time ray marching or volume rendering with high throughput. Unlike [Rasterization-based Voxelization](908-rasterization-based-voxelization.md), which projects triangles via the fixed-function graphics pipeline, compute shader methods decouple intersection logic from rasterization, allowing custom geometric tests and atomic operations on 3D buffers or textures.

### [GPU Voxelization](2633-gpu-voxelization.md)
GPU Voxelization converts a triangle mesh into a regular voxel grid by leveraging the GPU's hardware rasterization pipeline to map fragments directly to 3D grid cells. It solves the performance bottleneck of CPU-based ray-tracing voxelization by parallelizing the intersection test across the rasterizer. It differs from [3D Grid Rasterization](903-3d-grid-rasterization.md) by using a 3D texture UAV or SSBO as the direct write target rather than a 2D accumulation buffer, and from [GPU Volume Kernel Voxelization](1496-gpu-volume-kernel-voxelization.md) by relying on standard triangle rasterization rather than volume kernels.

### Mesh-to-Voxel Geometry Shader
The Mesh-to-Voxel Geometry Shader is a real-time DirectX technique that converts mesh objects into voxel-like cubes by iterating over integer grid positions within a triangle's bounding box and emitting explicit cube geometry for grid cells close to the triangle plane [S1]. Unlike rasterization-based voxelization which fills pixels or conservative voxelization which determines occupancy, this method explicitly generates the 3D geometry of every intersecting voxel, resulting in significantly higher vertex and triangle counts [S1].

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
Parametric Surface Voxelization converts a parametric surface (such as a subdivision surface or implicit field) into a voxel grid by rasterizing points in the 2D parameter space rather than the 3D object space. It differs from triangle-based rasterization methods by operating directly on the parametric domain, allowing for the voxelization of complex implicit or subdivision surfaces without explicit triangulation. This approach is particularly suited for real-time applications where rapid voxelization of non-polygonal geometry is required.

### [Position Map and Image-Space Transfer](6237-position-map-and-image-space-transfer.md)
This family of techniques converts 3D geometric data into voxel grids by projecting or transferring information into 2D image space (textures) before reconstructing the 3D volume. Unlike direct 3D rasterization, these methods leverage the GPU's fixed-function rasterizer or image load/store operations on 2D textures to determine voxel occupancy, often trading memory bandwidth for computational simplicity. They are distinguished from direct triangle rasterization by operating on pre-computed surface samples or 2D texture slices, ensuring connectivity or enabling real-time performance insensitive to scene complexity.

#### Rasterization-based Volume Modeling
The process of building volume data voxel-by-voxel, considered the third layer in the voxel modeling abstraction hierarchy [S5, unofficial]. It solves the problem of assigning density values to voxels based on primitive contributions. It differs from instantiation-based modeling by "pulling" values into a voxel (similar to raytracing), whereas instantiation-based modeling "pushes" values into voxels (similar to micropolygon rendering) [S5, unofficial]. The algorithm involves instancing the primitive, bounding it to find overlapping voxels, and sampling the density function at each voxel in the overlapping region [S5, unofficial]. The main loop iterates over voxels in the discrete bounding box, transforms the voxel position to world space, samples the primitive, and adds the value to the buffer if positive [S5, unofficial].

#### Single-Pass Voxelization
A voxelization approach that utilizes the geometry shader to reduce draw calls by selecting the axis from which each triangle has the largest overall projection for fragment generation [S6, unofficial]. It solves the problem of multiple draw calls required for multi-axis projection. It employs conservative rasterization to generate fragments as long as there is something inside the pixel, not only if its center is covered [S6, unofficial]. It uses an MSAA render target to solve the issue of incorrectly filling empty cells caused by conservative rasterization generating too many fragments [S6, unofficial]. Hardware support for conservative rasterization is available on Nvidia Maxwell architecture and later; RTX driver support is available for Turing and Pascal architecture GPUs [S6, unofficial]. A pitfall is that GPUs only generate fragments if a pixel's center is covered by a triangle, leading to small cracks at the edges of triangles if conservative rasterization is not used [S6, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N_{faces} \times N_{voxels\_per\_face})$, where $N_{voxels\_per\_face}$ depends on the triangle's bounding box size in voxel units [S5, unofficial].
*   **Memory Complexity:** $O(N_{occupied\_voxels})$ when using hash tables [S3, S4, unofficial].
*   **Performance:** CPU-based implementations can be slow for high-resolution grids due to sequential iteration [S1]. GPU acceleration (via compute shaders or hardware rasterization) provides significant speedups (e.g., 50x) [S1].
*   **Strengths:** Simple to implement; generates explicit surface geometry (cube meshes) suitable for voxel art or ray marching [S1][S2, unofficial]; sparse storage handles large bounding boxes efficiently [S3, unofficial].
*   **Limitations:** Output mesh size can be very large compared to the input (e.g., 100x vertex count increase) [S2, unofficial]; standard rasterization may produce holes without conservative techniques or precision tuning [S2, S6, unofficial].
*   **When to Use:** When an explicit voxel mesh (for rendering or further processing) is required from a triangle mesh, and the mesh is not extremely dense [S2, unofficial].
*   **When Not to Use:** When deep volumetric data (interior density) is needed, as this method typically produces shell voxelization [S2, unofficial]; when memory is extremely constrained and the mesh is very dense [S2, unofficial].

## Background
Originating from the need to convert polygonal models into voxel formats for real-time rendering and voxel art, this method is implemented in libraries such as Point-Cloud-Utils [S1], the C-based voxelizer library by Karim Naaji [S4, unofficial], and madcad [S3, unofficial]. It is widely used in voxel art pipelines, real-time ray tracing voxelization [S6, unofficial], and as a preprocessing step for volume rendering [S5, unofficial]. The approach leverages standard graphics pipeline concepts (rasterization) adapted for 3D grid occupancy.

## Sources
[S1] Voxelizing a Triangle Mesh - Point Cloud Utils, https://fwilliams.info/point-cloud-utils/sections/voxelizing_a_triangle_mesh/, fetched 2026-10-26
[S2] Mesh Voxelization - Geeks3D, https://www.geeks3d.com/20160531/mesh-voxelization/, fetched 2026-10-26 (unofficial)
[S3] In Python, how do I voxelize a 3D mesh - Stack Overflow, https://stackoverflow.com/questions/11851342/in-python-how-do-i-voxelize-a-3d-mesh, fetched 2026-10-26 (unofficial)
[S4] mesh voxelizer library, https://github.com/karimnaaji/voxelizer, fetched 2026-10-26 (unofficial)
[S5] Production Volume Rendering, https://cg.informatik.uni-freiburg.de/intern/seminar/volumeRenderingProductionFundamentals2011.pdf, fetched 2026-10-27 (unofficial)
[S6] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
