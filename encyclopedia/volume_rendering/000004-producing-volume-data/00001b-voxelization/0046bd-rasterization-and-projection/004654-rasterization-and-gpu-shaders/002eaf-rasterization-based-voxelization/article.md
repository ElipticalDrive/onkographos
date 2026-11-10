# Rasterization-based Voxelization

Also called: Cube Mesh Generation, Hash-based Surface Voxelization, Sparse Voxelization with PositionMap Hashing, Voxelizing a Triangle Mesh, madcad Voxelization

## Overview
Rasterization-based Voxelization converts a continuous triangle mesh into a discrete volumetric representation by projecting the geometry onto a 3D grid using the GPU's fixed-function rasterization pipeline [S1][S4, unofficial]. It solves the performance bottleneck of CPU-based geometric intersection tests by leveraging parallel fragment generation to determine voxel occupancy [S1][S3, unofficial]. Unlike [3D Grid Rasterization](903-3d-grid-rasterization.md), which projects onto orthogonal 2D planes and combines results, this method typically maps fragments directly to 3D grid cells or uses intermediate position maps to populate a sparse hash table [S3, S4, unofficial]. It differs from [GPU Volume Kernel Voxelization](1496-gpu-volume-kernel-voxelization.md) by relying on hardware rasterization rather than explicit compute-shader intersection logic [S1].

## How it works
**Inputs:**
*   Input mesh: Vertices $V$ and Faces $F$ (triangles).
*   Voxel size: $v_s = (v_{sx}, v_{sy}, v_{sz})$.
*   Voxel origin: $v_o = (o_x, o_y, o_z)$, the world-space coordinate of the $(0,0,0)$ voxel corner [S1][S4, unofficial].
*   Precision: $p$, a tolerance factor to reduce artifacts [S2, S4, unofficial].

**Outputs:**
*   Occupied voxel coordinates (integer indices $ijk$) [S1].
*   Optional: Cube mesh geometry (vertices, normals, indices) representing the voxelized surface [S4, unofficial].
*   Optional: Point cloud of voxel centers with colors [S4, unofficial].

**Data Structures:**
1.  **Hash Table (Sparse Storage):** Used to store occupied voxels to avoid dense 3D arrays.
    *   Key: Voxel center position or integer index $(i, j, k)$.
    *   Value: `voxel_data_t` containing world-space position and optional color [S4, unofficial].
    *   Implementation: Array of linked lists or buckets, hashed by position [S4, unofficial].
2.  **Output Mesh (Cube Mesh):**
    *   `vertices`: Array of size $N_{voxels} \times 8$.
    *   `normals`: Array of size 6 (predefined for cube faces).
    *   `indices`: Array of size $N_{voxels} \times N_{indices\_per\_voxel}$.
    *   `colors`: Array of size $N_{voxels} \times 8$ (optional) [S4, unofficial].

**Steps:**

1.  **Compute Half-Voxel Size:**
    Calculate the half-extents of a voxel:
    $$ h_v = v_s \times 0.5 $$
    $$ h_v = (h_{vx}, h_{vy}, h_{vz}) $$ [S4, unofficial]

2.  **Rasterize Triangles to Voxel Grid:**
    For each triangle in the mesh:
    a.  Determine the bounding box of the triangle in world space.
    b.  Map the bounding box to voxel integer coordinates.
    c.  Rasterize the triangle into the 3D grid. This can be done via:
        *   **Direct 3D Rasterization:** Using a 3D texture or volume buffer as the render target, writing depth/position to occupied cells [S1][S3, unofficial].
        *   **Position Map Hashing:** Projecting triangle points to a 2D position map (world-space coordinates) and hashing them to voxel keys [S3, unofficial].
        *   **Conservative Rasterization:** Ensuring all voxels intersected by the triangle are covered, not just those containing fragment centers [S6, unofficial].

3.  **Populate Hash Table:**
    For each occupied voxel identified in Step 2:
    a.  Compute the voxel center in world space:
        $$ c = (i + 0.5) \times v_s + v_o $$
    b.  Insert $(c, \text{color})$ into the hash table if not already present. The hash function maps the position to a bucket index [S4, unofficial].

4.  **Generate Output Geometry (Optional):**
    If a cube mesh is required:
    a.  Allocate output arrays for $N_{voxels}$ cubes.
    b.  Define the 8 corner offsets for a cube centered at $(0,0,0)$ using $h_v$:
        $$ \text{offsets} = \{ (\pm h_{vx}, \pm h_{vy}, \pm h_{vz}) \} $$ [S4, unofficial]
    c.  For each voxel in the hash table:
        i.  Retrieve its world-space center $c$ and color.
        ii. Generate 8 vertices by adding $c$ to each offset.
        iii. Assign predefined face normals (6 unique normals for a cube) [S4, unofficial].
        iv. Generate indices to form the 6 faces of the cube.
        v.  Copy color to the corresponding vertex slots.

5.  **Cleanup:**
    Free the hash table and intermediate structures [S4, unofficial].

**Pseudocode (Python-like, Hash-based approach):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rasterization_based_voxelization(mesh, voxel_size, voxel_origin, precision=0.01):
    """
    Voxelize a triangle mesh using a hash table to store occupied voxels.
    
    Args:
        mesh: Object with vertices (V) and faces (F).
        voxel_size: Tuple (sx, sy, sz) defining voxel dimensions.
        voxel_origin: Tuple (ox, oy, oz) defining the origin of voxel (0,0,0).
        precision: Float tolerance for intersection checks.
        
    Returns:
        occupied_voxels: Set of integer tuples (i, j, k).
        voxel_centers: List of world-space positions for each occupied voxel.
    """
    occupied_voxels = set()
    voxel_centers = []
    
    # Precompute half-voxel size
    hvs = [s * 0.5 for s in voxel_size]
    
    # Helper to convert world position to voxel index
    def world_to_voxel_index(pos):
        # pos is (x, y, z)
        # voxel_index = floor((pos - origin) / voxel_size)
        ix = int((pos[0] - voxel_origin[0]) / voxel_size[0])
        iy = int((pos[1] - voxel_origin[1]) / voxel_size[1])
        iz = int((pos[2] - voxel_origin[2]) / voxel_size[2])
        return (ix, iy, iz)

    # Helper to compute voxel center from index
    def voxel_index_to_center(idx):
        i, j, k = idx
        cx = (i + 0.5) * voxel_size[0] + voxel_origin[0]
        cy = (j + 0.5) * voxel_size[1] + voxel_origin[1]
        cz = (k + 0.5) * voxel_size[2] + voxel_origin[2]
        return (cx, cy, cz)

    # Rasterization Step: Iterate triangles and determine occupied voxels
    # Note: In a real GPU implementation, this is done via fragment shaders.
    # Here we simulate the logic of determining which voxels a triangle covers.
    for face in mesh.faces:
        # Get the 3 vertices of the triangle
        v0, v1, v2 = mesh.vertices[face[0]], mesh.vertices[face[1]], mesh.vertices[face[2]]
        
        # Determine the bounding box of the triangle in voxel space
        # This involves finding the min/max voxel indices covered by the triangle
        # For simplicity, we sample the triangle's vertices and edges, 
        # and use a conservative rasterization logic (e.g., checking all voxels 
        # whose centers are within the triangle's projected bounding box and 
        # inside the triangle plane).
        
        # Simulated rasterization: Collect all voxel indices intersected by the triangle
        # In practice, this uses the GPU's rasterizer or a conservative intersection test.
        intersected_voxels = rasterize_triangle_to_voxels(v0, v1, v2, voxel_size, voxel_origin, precision)
        
        for idx in intersected_voxels:
            if idx not in occupied_voxels:
                occupied_voxels.add(idx)
                voxel_centers.append(voxel_index_to_center(idx))
                
    return occupied_voxels, voxel_centers

def rasterize_triangle_to_voxels(v0, v1, v2, voxel_size, voxel_origin, precision):
    """
    Simulates the GPU rasterization step.
    Returns a set of voxel indices (i,j,k) that the triangle intersects.
    """
    # This function would typically be implemented on the GPU.
    # On CPU, it might use a bounding box traversal and plane intersection tests.
    # For the purpose of this article, we assume the existence of a robust 
    # rasterization primitive that returns all covered voxel indices.
    
    # Calculate bounding box of triangle in world space
    min_x = min(v0[0], v1[0], v2[0])
    max_x = max(v0[0], v1[0], v2[0])
    min_y = min(v0[1], v1[1], v2[1])
    max_y = max(v0[1], v1[1], v2[1])
    min_z = min(v0[2], v1[2], v2[2])
    max_z = max(v0[2], v1[2], v2[2])
    
    # Convert to voxel indices
    min_ix = int((min_x - voxel_origin[0]) / voxel_size[0])
    max_ix = int((max_x - voxel_origin[0]) / voxel_size[0])
    min_iy = int((min_y - voxel_origin[1]) / voxel_size[1])
    max_iy = int((max_y - voxel_origin[1]) / voxel_size[1])
    min_iz = int((min_z - voxel_origin[2]) / voxel_size[2])
    max_iz = int((max_z - voxel_origin[2]) / voxel_size[2])
    
    covered_voxels = set()
    
    # Iterate over the bounding box voxels
    for i in range(min_ix, max_ix + 1):
        for j in range(min_iy, max_iy + 1):
            for k in range(min_iz, max_iz + 1):
                # Check if the voxel center is inside the triangle
                # Or use a more robust conservative rasterization test
                center = voxel_index_to_center((i, j, k))
                if is_voxel_intersected_by_triangle(center, v0, v1, v2, precision):
                    covered_voxels.add((i, j, k))
                    
    return covered_voxels

def is_voxel_intersected_by_triangle(center, v0, v1, v2, precision):
    """
    Checks if a point (voxel center) is inside or very close to a triangle.
    Uses barycentric coordinates or plane distance.
    """
    # Simple point-in-triangle test using barycentric coordinates
    # This is a placeholder for the actual GPU rasterization logic.
    # In reality, the GPU rasterizer handles this efficiently.
    # We assume a conservative test that includes points within 'precision' distance.
    
    # Vector math to determine if the point is within the triangle
    # For brevity, we return True if the point is within the bounding box 
    # and satisfies a basic plane equation check.
    # A real implementation would use the specific rasterization algorithm 
    # (e.g., edge function evaluation).
    
    # Placeholder: Assume intersection if within a small epsilon of the triangle plane
    # and within the projected 2D bounds.
    return True # [G] Simplified for pseudocode completeness; real impl uses edge functions.
```

**Parameters:**
*   `voxel_size`: Typical values range from 0.1 to 1.0 units, depending on scene scale [S2, unofficial]. In Point-Cloud-Utils, a common value is $1/256.0$ [S1].
*   `voxel_origin`: Typically $(0,0,0)$ [S1].
*   `precision`: A tolerance factor to reduce "holes" artifacts. Typical value: $0.01$ [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Holes/Artifacts:** Standard rasterization only covers pixels/voxels whose centers lie inside the triangle, leading to cracks at edges. This is mitigated by using conservative rasterization or a precision factor [S2, S6, unofficial].
*   **Performance:** The output mesh size scales with voxel density. A torus mesh (441 vertices, 800 faces) with voxel size 0.75 results in 61,312 vertices and 91,968 faces [S2, unofficial].
*   **Memory:** Using a hash table (sparse voxelization) is more memory-efficient for sparse meshes than dense 3D arrays [S3, unofficial].

## Subtopics

### [3D Grid Rasterization](903-3d-grid-rasterization.md)
Converts a triangle mesh into a discrete 3D occupancy grid by projecting the mesh onto three orthogonal planes (XY, YZ, XZ) using hardware rasterization and combining the results. It differs from CPU-based bounding-box intersection methods by leveraging GPU fragment shaders to determine voxel occupancy, and from full surface reconstruction methods (like o-voxel) by outputting only binary presence without solving for dual vertices or QEF. It is a rasterization-based voxelization technique that produces a dense or sparse grid depending on the storage format used.

### [GPU Volume Kernel Voxelization](1496-gpu-volume-kernel-voxelization.md)
Accelerates the voxelization of triangle meshes by parallelizing the iteration over the 2D XY plane of the voxel grid using a GPU compute shader. It solves the performance bottleneck of CPU-based voxelization, where nested loops over X and Y coordinates are executed sequentially. Unlike [Rasterization-based Voxelization](908-rasterization-based-voxelization.md) and [3D Grid Rasterization](903-3d-grid-rasterization.md) which project geometry into 3D textures, this method retains the explicit triangle-voxel intersection logic of CPU implementations but distributes the workload across GPU threads, achieving significant speedups (e.g., 50x) for high-resolution grids.

### [GPU Voxelization via Compute Shaders](2549-gpu-voxelization-via-compute-shaders.md)
Converts 3D triangle meshes into a uniform 3D voxel grid by leveraging GPU parallel processing. It solves the problem of generating volumetric data for real-time ray marching or volume rendering with high throughput. Unlike [Rasterization-based Voxelization](908-rasterization-based-voxelization.md), which projects triangles via the fixed-function graphics pipeline, compute shader methods decouple intersection logic from rasterization, allowing custom geometric tests and atomic operations on 3D buffers or textures.

### [GPU Voxelization](2633-gpu-voxelization.md)
Converts a triangle mesh into a regular voxel grid by leveraging the GPU's hardware rasterization pipeline to map fragments directly to 3D grid cells. It solves the performance bottleneck of CPU-based ray-tracing voxelization by parallelizing the rasterization process across the GPU. It differs from [3D Grid Rasterization](903-3d-grid-rasterization.md) by writing directly to a 3D texture UAV or SSBO instead of a 2D accumulation buffer.

### [CPU Mesh Voxelization](3137-cpu-mesh-voxelization.md)
Converts a triangle mesh into a 3D grid of voxels by iterating over the bounding box of each triangle and testing each voxel center against the triangle's plane and edge constraints. It differs from its parent [Rasterization-based Voxelization](908-rasterization-based-voxelization.md) by outputting a binary occupancy grid or bit-packed table rather than generating a cube mesh per occupied voxel. It differs from [3D Grid Rasterization](903-3d-grid-rasterization.md) by using exact geometric overlap tests (SAT or plane/projection) rather than simple grid cell intersection, and from [Voxelization via Triangle Scattering](3779-voxelization-via-triangle-scattering.md) by iterating cells against triangles rather than scattering triangles into cells.

### [Voxel Rasterization](3204-voxel-rasterization.md)
Converts a triangle mesh into a voxelized representation by rasterizing triangles directly into a 3D voxel grid. It differs from its parent [Rasterization-based Voxelization](908-rasterization-based-voxelization.md) by focusing on the specific algorithmic steps of iterating over triangle surfaces, interpolating normals/colors, and writing to a voxel buffer, often leveraging GPU acceleration for performance. Unlike [3D Grid Rasterization](903-3d-grid-rasterization.md) which may use different traversal strategies, this method explicitly samples along triangle edges and spans to fill voxels.

### [Conservative Voxelization](3247-conservative-voxelization.md)
Hardware-accelerated algorithm for polygonal models that guarantees conservative correctness: every voxel intersecting the input model is correctly recognized. It differs from standard rasterization-based voxelization [Rasterization-based Voxelization] by computing the exact depth range of the intersection between the triangle and the voxel column, rather than using a single depth value which leads to missed voxels.

### [Conservative Rasterization via Triangle Expansion](3531-conservative-rasterization-via-triangle-expansion.md)
Eliminates voxelization holes caused by standard rasterization, which only covers pixels whose centers lie inside a triangle, by expanding the triangle in clip space and culling the expanded area in the fragment shader. It differs from sibling methods like [Conservative Voxelization](3247-conservative-voxelization.md) by implementing the expansion logic manually in the geometry shader using homogeneous plane mathematics, rather than relying on hardware-specific conservative rasterization extensions or post-processing hole-filling.

### [Multi-view projection](5937-multi-view-projection.md)
Multi-view projection voxelization converts a triangle mesh into a voxel grid by rasterizing the geometry from multiple camera viewpoints, typically aligned with the principal axes of the scene's bounding box. This approach solves the coverage gaps and geometric distortions inherent in single-view projection methods, which fail when surface normals are perpendicular to the view direction. It differs from single-view methods by ensuring comprehensive coverage through redundancy, and from compute-shader-based methods by leveraging standard graphics pipeline rasterization.

### [Position Map and Image-Space Transfer](6237-position-map-and-image-space-transfer.md)
Position Map and Image-Space Transfer is a family of rasterization-based voxelization techniques that decouple surface intersection from volume grid population by using intermediate 2D image textures (position maps) to store world-space surface coordinates. It solves the problem of efficiently mapping continuous geometric surfaces to discrete voxel grids by leveraging the GPU's rasterization pipeline to compute visibility and position, rather than performing geometric clipping or ray-marching against the volume grid directly. It differs from direct triangle rasterization into a voxel grid by operating on pre-computed surface samples in image space, and from hardware-accelerated surface voxelization via 2D textures by explicitly using a 2D image store to write directly into a 3D volume texture during a second pass.

### Rasterization-based Volume Modeling
The process of building volume data voxel-by-voxel, considered the third layer in the voxel modeling abstraction hierarchy [S5, unofficial]. It involves instancing the primitive representation, bounding it to find the region of voxels it overlaps, and sampling its density function at each voxel in the overlapping region [S5, unofficial]. The main rasterization loop iterates over voxels in the discrete bounding box, transforms voxel position to world space, samples the primitive, and adds the value to the buffer if positive [S5, unofficial]. Rasterization-based modeling pulls values into a voxel (similar to raytracing), whereas instantiation-based modeling pushes values into voxels (similar to micropolygon rendering) [S5, unofficial].

### Single-Pass Voxelization
A voxelization approach that utilizes the geometry shader to reduce draw calls by selecting the axis from which each triangle has the largest overall projection for fragment generation [S6, unofficial]. It employs conservative rasterization to generate fragments as long as there is something inside the pixel, not only if its center is covered [S6, unofficial]. It uses an MSAA render target during the voxelization process to solve the issue of incorrectly filling empty cells [S6, unofficial]. Hardware support for conservative rasterization is available on Nvidia Maxwell architecture and later; RTX driver support is available for Turing and Pascal architecture GPUs [S6, unofficial]. A pitfall is that GPUs only generate fragments if a pixel's center is covered by a triangle, leading to small cracks at the edges of triangles, and conservative rasterization can generate too many fragments, incorrectly filling cells in the voxel grid that should be empty [S6, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot R)$, where $N$ is the number of triangles and $R$ is the average number of voxels covered per triangle. The rasterization step is highly parallelized on the GPU, making it significantly faster than CPU-based methods for high-resolution grids [S1][S5, unofficial].
*   **Memory Complexity:** $O(V)$, where $V$ is the number of occupied voxels. Using a hash table (sparse voxelization) reduces memory usage compared to dense 3D arrays, especially for sparse meshes [S3, unofficial].
*   **Measured Performance:** For a torus mesh (441 vertices, 800 faces), voxel size 0.75 results in 61,312 vertices and 91,968 faces; voxel size 0.5 results in 144,968 vertices and 217,452 faces [S2, unofficial]. The output mesh size scales with voxel density, impacting rendering performance if not managed [S2, unofficial].
*   **Strengths:** High performance due to GPU parallelization; simple implementation using standard graphics pipeline; good for shell voxelization (surface only) [S2, unofficial].
*   **Limitations:** Can produce holes or artifacts if not using conservative rasterization; output mesh size can be large; does not perform deep voxelization (interior voxels) by default [S2, S6, unofficial].
*   **When to Use:** Real-time applications requiring fast voxelization of triangle meshes; voxel art generation; preprocessing for volume rendering or ray marching [S1][S2, unofficial].
*   **When Not to Use:** When deep voxelization (interior) is required; when exact geometric intersection is critical without approximation; when memory is extremely constrained and sparse storage is not feasible [S2, S3, unofficial].

## Background
Rasterization-based voxelization originated as a method to leverage the GPU's fixed-function pipeline for volume data generation. It is implemented in libraries such as Point-Cloud-Utils [S1] and the C-based voxelizer library by Karim Naaji [S4, unofficial]. The technique is widely used in voxel art tools and real-time rendering pipelines [S2, unofficial]. It is considered a "pull" based approach in volume modeling, where values are pulled into a voxel, contrasting with "push" based instantiation methods [S5, unofficial].

## Sources
[S1] Voxelizing a Triangle Mesh - Point Cloud Utils, https://fwilliams.info/point-cloud-utils/sections/voxelizing_a_triangle_mesh/, fetched 2026-10-26
[S2] Mesh Voxelization - Geeks3D, https://www.geeks3d.com/20160531/mesh-voxelization/, fetched 2026-10-26 (unofficial)
[S3] In Python, how do I voxelize a 3D mesh - Stack Overflow, https://stackoverflow.com/questions/11851342/in-python-how-do-i-voxelize-a-3d-mesh, fetched 2026-10-26 (unofficial)
[S4] mesh voxelizer library, https://github.com/karimnaaji/voxelizer, fetched 2026-10-26 (unofficial)
[S5] Production Volume Rendering, https://cg.informatik.uni-freiburg.de/intern/seminar/volumeRenderingProductionFundamentals2011.pdf, fetched 2026-10-27 (unofficial)
[S6] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
