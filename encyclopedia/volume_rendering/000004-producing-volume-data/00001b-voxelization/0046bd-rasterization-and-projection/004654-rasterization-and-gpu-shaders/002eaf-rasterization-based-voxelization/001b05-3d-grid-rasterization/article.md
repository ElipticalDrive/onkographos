# 3D Grid Rasterization

Also called: Grid-Snapped Voxelization

## Overview
3D Grid Rasterization converts a triangle mesh into a binary occupancy grid by projecting the mesh onto three orthogonal planes (XY, YZ, XZ) using hardware rasterization and combining the results [S4, unofficial]. It solves the problem of efficiently determining which voxels in a fixed-resolution 3D grid are intersected by the mesh surface without performing explicit CPU-side raycasting or complex geometric intersection tests [S4, unofficial]. It differs from [Conservative Voxelization](3247-conservative-voxelization.md) by relying on standard rasterization rules (which may miss thin features) and from [A-Buffer Solid Voxelization](1189-a-buffer-solid-voxelization.md) by discarding occluded fragments rather than retaining depth lists, making it faster but potentially less watertight for complex internal structures [S1, S4, unofficial].

## How it works
**Inputs:** A triangle mesh (vertices, indices, optional colors/normals), a target resolution (integer), and grid bounds.
**Outputs:** A 3D occupancy grid (binary or colored) representing the voxelized mesh.

**Data Structures:**
*   **Voxel Grid Storage:** A compact 1-bit-per-voxel format. For a grid of dimensions $N_x \times N_y \times N_z$, the storage is a `std::vector<uint32_t>` of size $\lceil (N_x \cdot N_y \cdot N_z) / 32 \rceil$. Each `uint32_t` stores 32 voxels. Memory order is linear in X, then Y, then Z [S4, unofficial].
*   **Projection Textures:** Three 3D textures (`xProjTextureId`, `yProjTextureId`, `zProjTextureId`) used as intermediate buffers. Their dimensions are adjusted to store 32 slices per texture layer to match the bit-packing width. For example, `xProjTextureId` has dimensions $(N_y, N_z, N_x/32)$ [S4, unofficial].
*   **VoxelCoord:** A struct with integer fields `x`, `y`, `z`. Hash function for sparse variants uses primes $p_1=73856093, p_2=19349663, p_3=83492791$: $hash = x \cdot p_1 \oplus y \cdot p_2 \oplus z \cdot p_3$ [S1, unofficial].

**Parameters:**
*   `resolution`: Target grid resolution along the smallest bounding box dimension. Typical values: 64, 128, 256, 512 [S4, unofficial].
*   `voxel_size`: Calculated as $\min(\text{boundingBoxSize}) / \text{resolution}$ [S4, unofficial].
*   `grid_min`, `grid_max`: Integer bounds of the grid, typically $\{0,0,0\}$ and $\{\text{grid\_size}, \text{grid\_size}, \text{grid\_size}\}$ [S1, unofficial].

**Steps:**

1.  **Grid Initialization:**
    *   Compute the mesh bounding box in world space.
    *   Calculate `voxel_size` based on the smallest dimension of the bounding box and the target `resolution` [S4, unofficial].
    *   Round the number of voxels in each dimension up to the nearest multiple of 32 to facilitate bit-packing [S4, unofficial].
    *   Calculate `minCorner` and `maxCorner` based on the bounding box center and the rounded voxel count:
        $$ \text{minCorner} = \text{boundingBoxCenter} - 0.5 \times (\text{roundedNbVoxels} \times \text{voxelSize}) $$
        [S4, unofficial]
    *   Allocate the three projection 3D textures and the final compact 1-bit grid [S4, unofficial].

2.  **X-Axis Projection (YZ Plane):**
    *   Iterate through the X-axis in steps of 32 (slabs). For each slab index `sliceX`:
        *   Bind the corresponding slice of `xProjTextureId` to the framebuffer.
        *   Set viewport to $(N_y, N_z)$.
        *   Clear the buffer.
        *   Set up an orthographic projection matrix looking down the X-axis. The view frustum covers the Y and Z extents of the grid, and the near/far planes cover the current 32-voxel slab in X:
            $$ \text{viewProj} = \text{ortho}(\min Y, \max Y, \min Z, \max Z, \text{sliceX} \cdot \text{voxelSize}, (\text{sliceX} + 32) \cdot \text{voxelSize}) $$
            [S4, unofficial]
        *   Apply rotation to align the mesh for X-projection: $R = \text{rotate}(-90^\circ, \hat{x}) \times \text{rotate}(90^\circ, \hat{z})$ [S4, unofficial].
        *   Rasterize the mesh triangles into the 2D slice. Use `GL_OR` logic operation or additive blending to set bits in the texture [S4, unofficial].

3.  **Y-Axis Projection (XZ Plane):**
    *   Iterate through the Y-axis in steps of 32. For each `sliceY`:
        *   Bind slice of `yProjTextureId` (dimensions $N_x \times N_z \times N_y/32$).
        *   Set viewport to $(N_x, N_z)$.
        *   Orthographic projection:
            $$ \text{viewProj} = \text{ortho}(\min X, \max X, \min Z, \max Z, \text{sliceY} \cdot \text{voxelSize}, (\text{sliceY} + 32) \cdot \text{voxelSize}) $$
            [S4, unofficial]
        *   Rotation: $R = \text{rotate}(-90^\circ, \hat{x})$ [S4, unofficial].
        *   Rasterize mesh into the slice.

4.  **Z-Axis Projection (XY Plane):**
    *   Iterate through the Z-axis in steps of 32. For each `sliceZ`:
        *   Bind slice of `zProjTextureId` (dimensions $N_x \times N_y \times N_z/32$).
        *   Set viewport to $(N_x, N_y)$.
        *   Orthographic projection:
            $$ \text{viewProj} = \text{ortho}(\min X, \max X, \min Y, \max Y, \text{sliceZ} \cdot \text{voxelSize}, (\text{sliceZ} + 32) \cdot \text{voxelSize}) $$
            [S4, unofficial]
        *   Rotation: $R = \text{rotate}(180^\circ, \hat{z}) \times \text{rotate}(180^\circ, \hat{y})$ [S4, unofficial].
        *   Rasterize mesh into the slice.

5.  **Combination (Compilation):**
    *   Use a `compileProjections` shader. Bind `xProjTextureId` to unit 0, `yProjTextureId` to unit 1.
    *   For each `sliceZ` (step 32):
        *   Set uniform `slice` to `sliceZ`.
        *   Bind the target slice of `zProjTextureId` to the framebuffer.
        *   Draw a full-screen quad (or triangle strip) to combine the X and Y projections into the Z texture. The fragment shader computes the logical AND of the bits from the X and Y projections to determine final occupancy for the Z-slab [S4, unofficial].
        *   *Note:* Some implementations use a single pass per axis and combine later, or use a 3D texture with image stores for direct writing [S3, S4, unofficial]. The specific "compile" step described in [S4, unofficial] combines X and Y to verify Z-intersection.

6.  **Result Retrieval:**
    *   Read the final `zProjTextureId` (or the combined result) using `glGetTexImage` with format `GL_RGBA_INTEGER` and type `GL_UNSIGNED_INT_8_8_8_8` (or similar bit-packed format) into a CPU buffer [S4, unofficial].
    *   Unpack the bits into the final 1-bit grid or convert to a boolean array [S4, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rasterize_3d_grid(mesh, resolution):
    # 1. Compute Grid Parameters
    bbox_min, bbox_max = mesh.compute_bounding_box()
    bbox_size = bbox_max - bbox_min
    min_dim = min(bbox_size.x, bbox_size.y, bbox_size.z)
    voxel_size = min_dim / resolution
    
    # Round dimensions to multiple of 32
    nb_voxels = [
        ceil(bbox_size.x / voxel_size) / 32 * 32,
        ceil(bbox_size.y / voxel_size) / 32 * 32,
        ceil(bbox_size.z / voxel_size) / 32 * 32
    ]
    
    # Center the grid
    bbox_center = (bbox_min + bbox_max) / 2.0
    min_corner = bbox_center - 0.5 * (nb_voxels * voxel_size)
    
    # 2. Allocate Textures
    x_proj = allocate_3d_texture(nb_voxels.y, nb_voxels.z, nb_voxels.x / 32)
    y_proj = allocate_3d_texture(nb_voxels.x, nb_voxels.z, nb_voxels.y / 32)
    z_proj = allocate_3d_texture(nb_voxels.x, nb_voxels.y, nb_voxels.z / 32)
    
    # 3. X-Axis Projection
    for slice_x in range(0, int(nb_voxels.x), 32):
        bind_texture_3d_slice(x_proj, slice_x / 32)
        set_viewport(nb_voxels.y, nb_voxels.z)
        clear_buffer()
        
        # Ortho projection for X-slab
        view_proj = ortho(
            min_corner.y, min_corner.y + nb_voxels.y * voxel_size,
            min_corner.z, min_corner.z + nb_voxels.z * voxel_size,
            min_corner.x + slice_x * voxel_size,
            min_corner.x + (slice_x + 32) * voxel_size
        )
        rotation = rotate(-90, vec3(1,0,0)) * rotate(90, vec3(0,0,1))
        draw_mesh(mesh, view_proj, rotation)
        
    # 4. Y-Axis Projection
    for slice_y in range(0, int(nb_voxels.y), 32):
        bind_texture_3d_slice(y_proj, slice_y / 32)
        set_viewport(nb_voxels.x, nb_voxels.z)
        clear_buffer()
        
        view_proj = ortho(
            min_corner.x, min_corner.x + nb_voxels.x * voxel_size,
            min_corner.z, min_corner.z + nb_voxels.z * voxel_size,
            min_corner.y + slice_y * voxel_size,
            min_corner.y + (slice_y + 32) * voxel_size
        )
        rotation = rotate(-90, vec3(1,0,0))
        draw_mesh(mesh, view_proj, rotation)
        
    # 5. Z-Axis Projection
    for slice_z in range(0, int(nb_voxels.z), 32):
        bind_texture_3d_slice(z_proj, slice_z / 32)
        set_viewport(nb_voxels.x, nb_voxels.y)
        clear_buffer()
        
        view_proj = ortho(
            min_corner.x, min_corner.x + nb_voxels.x * voxel_size,
            min_corner.y, min_corner.y + nb_voxels.y * voxel_size,
            min_corner.z + slice_z * voxel_size,
            min_corner.z + (slice_z + 32) * voxel_size
        )
        rotation = rotate(180, vec3(0,0,1)) * rotate(180, vec3(0,1,0))
        draw_mesh(mesh, view_proj, rotation)
        
    # 6. Combine Projections (Simplified: Logical AND of X, Y, Z)
    # In the specific implementation [S4, unofficial], X and Y are combined into Z.
    # Here we assume a generic combination step.
    final_grid = combine_projections(x_proj, y_proj, z_proj)
    
    return final_grid
```

## Subtopics

### [A-Buffer Solid Voxelization](1189-a-buffer-solid-voxelization.md)
A-Buffer Solid Voxelization is a GPU-based technique that voxelize a watertight mesh by rasterizing it into a fixed-width, height, and depth 3D grid and resolving multiple depth intersections per pixel using an A-buffer linked list structure. It differs from standard depth-buffer rasterization by retaining all fragment depth information rather than discarding occluded fragments, allowing it to output a linked list of starting and ending voxels for each row and column in the grid.

### [Adaptive Sparse Voxelization with Bounding Box Thickness Cases](1686-adaptive-sparse-voxelization-with-bounding-box-thickness-cas.md)
This method rasterizes triangle meshes into a Morton-ordered sparse voxel grid by classifying each triangle into one of nine cases based on the thickness of its axis-aligned bounding box (AABB) in grid coordinates. It solves the problem of efficiently determining intersected voxels while minimizing redundant depth tests, setting itself apart from general 3D grid rasterization by explicitly handling "thin" (1-voxel thick) triangles via 2D projection and "thick" triangles via 3D plane and projection tests.

### [Geometry Shader Voxelization](2566-geometry-shader-voxelization.md)
Geometry Shader Voxelization rasterizes polygonal meshes directly into a 3D voxel grid by leveraging the GPU's geometry shader stage to project triangles onto axis-aligned planes and iterate through occupied grid cells. It solves the problem of converting continuous surface geometry into discrete volumetric data without requiring explicit CPU-side raycasting or compute-shader-based brute-force intersection tests. Unlike sibling methods that rely on hardware rasterization of 2D fragments (e.g., [Image-space Voxelization](14395-image-space-voxelization.md)) or pure compute shaders (e.g., [GPU Mesh Voxelization via AABB-Triangle Intersection](2541-gpu-mesh-voxelization-via-aabb-triangle-intersection.md)), this method explicitly generates 3D voxel primitives or iterates grid bounds within the shader, offering a balance between implementation simplicity and watertightness control.

### [GPU Mesh Voxelizer (Grid Generation)](11413-gpu-mesh-voxelizer-grid-generation.md)
This method generates a dense 3D voxel grid by computing the center coordinates of each voxel cell directly on the GPU using a compute shader. It solves the problem of converting a mesh's spatial extent into a fixed-width, height, and depth 3D grid structure, returning a flat array of voxel centers. Unlike rasterization-based approaches that project geometry onto slices, this method calculates grid dimensions from the mesh's bounding box and fills the grid with uniform cells, making it suitable for preprocessing steps where a regular grid topology is required. It builds on [3D Grid Rasterization](903-3d-grid-rasterization.md) by providing the specific grid generation mechanism.

### [Derived Grid Rasterization](11477-derived-grid-rasterization.md)
Derived Grid Rasterization converts an unstructured 3D volume mesh (e.g., tetrahedra, hexahedra) into a dense, regular 3D voxel grid spanning the mesh's Axis-Aligned Bounding Box (AABB). This creates a structured data layout required for efficient volume ray-marching and axis-aligned slice planes on data that originally lacks a grid structure. Unlike sibling methods that may produce sparse structures or rely on different rasterization primitives, this method explicitly maps each voxel to its owning unstructured cell and local tetrahedron index, enabling direct lookup during rendering. It builds on [3D Grid Rasterization](903-3d-grid-rasterization.md) by handling unstructured cell topology rather than simple surface mesh voxelization.

### [Unstructured Mesh Pixelization via Convex Element Containment](11479-unstructured-mesh-pixelization-via-convex-element-containmen.md)
A pixelization routine that transforms variable-resolution unstructured mesh values into an image buffer by depositing values through nearest-neighbor interpolation with anti-aliasing, decoupling the pixelization step from the underlying data model. It differs from standard rasterization by using a geometric containment test based on dot product signs relative to element centroids, which eliminates the requirement for consistent vertex winding order (CW/CCW).

### [Fast Solid Voxelization via Planar Raycasting](13748-fast-solid-voxelization-via-planar-raycasting.md)
Fast Solid Voxelization via Planar Raycasting is a rasterization-based method that generates a solid voxel mesh from an arbitrary polyhedron by casting rays along the three world coordinate planes (XY, YZ, XZ). It solves the problem of converting polygonal meshes into dense 3D grids by determining voxel occupancy through intersection parity, prioritizing raw computational speed over memory efficiency. It differs from its sibling [A-Buffer Solid Voxelization](1189-a-buffer-solid-voxelization.md) and [3D Grid Rasterization](903-3d-grid-rasterization.md) by using pre-computed plane buffers and bit-shifting accumulation rather than per-voxel raycasting or hierarchical data structures, making it significantly faster but less accurate for complex internal structures.

### [Image-space Voxelization](14395-image-space-voxelization.md)
Image-space voxelization projects the vertices of a triangular mesh into a 2D image stored in a Frame Buffer Object (FBO) to determine voxel locations and colors. It solves the problem of converting boundary models into volumetric representations by leveraging GPU rasterization hardware, specifically using a single render pass and a single buffer to preserve original vertex colors, unlike methods requiring multiple passes or z-buffer stacks.

### Projection Voxelization Shader
A vertex shader that generates a full-screen quad to process each voxel slice, outputting non-normalized voxel coordinates for the fragment shader [S4, unofficial]. It defines quad corners `vec2(0,0), vec2(1,0), vec2(0,1), vec2(1,1)`, calculates voxel coordinate `texelCoord = vec3(corner * gridSize.xy, slice)`, and sets clip space position `gl_Position = vec4(corner*2-1, 0, 1)` [S4, unofficial]. Input uniforms are `slice` (current Z-slice index) and `gridSize` (voxel grid dimensions) [S4, unofficial].

### 3D Rasterization Algorithm
A simplicial covering method that computes the minimal bounding box and centroid of the polygon/solid, constructs a tetrahedron (in 3D) or triangle (in 2D) between the origin vertex (centroid) and one edge (or face), and rasterizes the simplex in the P-Buffer (Presence Buffer), flipping all positions (bits) covered by it [S12, unofficial]. It repeats this for all edges/faces until no edges are left, then transfers all positions from the P-Buffer within the minimal bounding box that have a presence value of 1 to the framebuffer, applying the corresponding color [S12, unofficial]. The P-Buffer is a 3D buffer where each position stores a single bit representing the presence value of a voxel (1 if inside, 0 if outside) [S12, unofficial].

## Operating characteristics
**Time Complexity:** $O(N \cdot V)$ where $N$ is the number of triangles in the mesh and $V$ is the number of slices (proportional to grid resolution). Each slice renders the entire mesh [S4, unofficial].
**Memory Complexity:** $O(N_x \cdot N_y \cdot N_z / 32)$ for the compact 1-bit grid storage [S4, unofficial]. Intermediate 3D textures require similar memory proportional to the grid volume.
**Measured Performance:** On an Intel HD 4400 with a Suzanne model (31,000 vertices): $64^3$ grid takes 2 ms; $128^3$ takes 9 ms; $256^3$ takes 10 ms; $512^3$ takes 120 ms. A 1,000,000 vertex model at $128^3$ takes 45 ms [S4, unofficial]. On a CPU (Intel Xeon) vs GPU (NVIDIA A100) for a PegInsertionSide-v1 mesh (281K verts, 561K faces) at $1024^3$ resolution: CPU takes 1.35s, GPU takes 0.56s (3x faster) [S1, unofficial].
**Strengths:** Leverages highly optimized GPU hardware rasterization; compact memory storage; simple implementation using standard OpenGL/Direct3D features.
**Limitations:** Precision glitches on some hardware (e.g., Linux/Intel HD 4400); triangles outside the grid bounds are not voxelized; thin features may be missed due to rasterization gaps; additive blending assumes opaque mesh [S3, S4, S5, unofficial].
**When to use:** When a fast, approximate occupancy grid is needed for global illumination (VXGI) or collision detection, and watertightness is not critical.
**When not to use:** When exact surface reconstruction (dual vertices) is required, or when the mesh has transparency or complex internal structures that require depth sorting [S1, S3, unofficial].

## Background
Originating from hardware-accelerated voxelization techniques for global illumination (VXGI) and real-time rendering, this method utilizes the GPU's rasterization pipeline to project geometry onto 3D grids [S3, S4, unofficial]. It is a core component of the o-voxel pipeline, specifically the "intersect scanline" step, which discovers occupied voxels without performing the subsequent QEF (Quadratic Error Function) minimization or dual vertex solving steps [S1, unofficial]. The technique was popularized by implementations such as TRELLIS.2 and various GPU voxelizer projects [S1, S4, unofficial].

## Sources
[S1] yuyujunjun/mesh-voxelize, https://github.com/yuyujunjun/mesh-voxelize, fetched 2026-10-26 (unofficial)
[S2] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
[S3] BoyBaykiller/IDKEngine, https://github.com/BoyBaykiller/IDKEngine, fetched 2026-10-26 (unofficial)
[S4] piellardj/voxelizer-gpu, https://github.com/piellardj/voxelizer-gpu, fetched 2026-10-26 (unofficial)
[S5] 3D-Mesh-Voxelizer - GitHub, https://github.com/Talkative-Banana/3D-Mesh-Voxelizer, fetched 2026-10-26 (unofficial)
[S6] Voxelization — Open3D 0.12.0 documentation, https://www.open3d.org/docs/0.12.0/tutorial/geometry/voxelization.html, fetched 2026-10-26
[S7] latencyhiding/voxel_cone_tracing, https://github.com/latencyhiding/voxel_cone_tracing, fetched 2026-10-26 (unofficial)
[S8] mesh voxelizer library, https://github.com/karimnaaji/voxelizer, fetched 2026-10-26 (unofficial)
[S9] Real-Time GPU-Based Voxelizer - GitHub Pages, https://freder.github.io/UnityGraphicsProgrammingBook1/html-translated/vol2/Chapter%201%20_%20Real-Time%20GPU-Based%20Voxelizer.html, fetched 2026-10-27
[S10] voxelizer-rs, https://github.com/lhemerly/voxelizer-rs, fetched 2026-10-28 (unofficial)
[S11] GitHub - Septaris/mesh_vox, https://github.com/Septaris/mesh_vox, fetched 2026-10-30 (unofficial)
[S12] Voxelization of solids using simplicial coverings, http://wscg.zcu.cz/wscg2004/Papers_2004_Short/J53.pdf, fetched 2026-10-26 (unofficial)
