# Geometry Shader Voxelization

Also called: GS Voxelization, Solid Triangle Voxelization, Triangle-to-Voxel Rasterization, Triangle Projection Voxelization, OpenGL Geometry Shader Voxelization

## Overview
Geometry Shader Voxelization converts polygonal meshes (`.obj`, `.ply`) into a discrete 3D occupancy or color grid by rasterizing triangles directly into a 3D texture using an OpenGL Geometry Shader [S3]. It solves the problem of efficiently filling a 3D grid using standard GPU hardware rasterizers, which are optimized for 2D screens [S4]. Unlike [GPU Mesh Voxelization via Axis-Aligned Projection](2561-gpu-mesh-voxelization-via-axis-aligned-projection.md) which explicitly selects the optimal projection axis per triangle to maximize coverage, this method uses generic 3D rasterization logic or swizzles geometry to align with the XY plane while checking overlaps in all three dimensions (XY, YZ, ZX) [S3]. It differs from [GPU Voxelization Using OpenGL Image Atomics](2544-gpu-voxelization-using-opengl-image-atomics.md) by typically relying on the geometry shader's ability to emit multiple primitives or iterate through grid cells, rather than pure atomic accumulation without depth-based projection [S4].

## How it works
**Inputs:**
- Mesh vertices $V$ (3xN floats) and faces $F$ (3xM uint32s) loaded from `.obj` or `.ply` files [S3].
- Voxel resolution `dim` (e.g., 64), applied uniformly to x, y, z axes [S3].
- Thickness mode: `THIN` (0) or `FAT` (1) [S3].

**Outputs:**
- Occupancy Grid: A 3D texture (`GL_TEXTURE_3D`) with internal format `GL_R8UI` (8-bit unsigned integer, single channel) [S3].
- Voxel Color: A 3D image `voxelColor` with format `rgba8` storing color data [S3].

**Data Structures:**
- `mesh.V`: Eigen matrix, size 3xN, float [S3].
- `mesh.F`: Eigen matrix, size 3xM, uint32 [S3].
- `voxelResolution`: Uniform `ivec3` [S3].
- `unswizzle`: Permutation matrix to reverse coordinate transformation [S3].

**Steps:**

1. **Load and Normalize Mesh:**
   Load vertices and faces. Compute bounding box: $xmin, xmax, ymin, ymax, zmin, zmax$. Calculate dimensions $dx, dy, dz$ and $dmax = \max(dx, dy, dz)$ [S3]. Normalize vertices to $[0, 1]$ range:
   $$ V_{norm} = \frac{V - min}{dmax} $$
   [S3].

2. **Setup Rendering Pipeline:**
   Load shaders (`VoxelizationVS.glsl`, `VoxelizationGS.glsl`, `VoxelizationFS.glsl`) [S3]. Bind vertex buffer `mesh.V` and element buffer `mesh.F`. Pass `voxelResolution` uniform [S3]. Bind `voxelOccupancy` (`layout(r8ui, binding = 0) uniform uimage3D`) and `voxelColor` (`layout(rgba8, binding = 1) uniform image3D`) [S3].

3. **Swizzle Triangle:**
   For each triangle, compute normal $n = \text{cross}(v_1 - v_0, v_2 - v_1)$ [S3]. Determine the dominant axis of the normal (max component of $|n|$) [S3]. Permute vertex coordinates and normal to align the dominant plane with the XY plane. Store the permutation matrix in `unswizzle` from `unswizzleLUT` to reverse the transformation later [S3].

4. **Compute Edge Normals:**
   For each edge ($e_0=v_1-v_0, e_1=v_2-v_1, e_2=v_0-v_2$), compute inward-facing 2D edge normals for the XY, YZ, and ZX planes. The direction depends on the sign of the normal component perpendicular to the plane [S3].
   Formula for XY edge normal $e_0$:
   $$ n_{e0,xy} = (n.z \ge 0) ? \text{vec2}(-e_0.y, e_0.x) : \text{vec2}(e_0.y, -e_0.x) $$
   [S3]. Similar logic applies to YZ and ZX planes using $n.x$ and $n.y$ respectively [S3].

5. **Calculate Initial Edge Distances:**
   Compute edge distances for the projected vertices.
   For **Thin** voxelization [S3]:
   $$ d_e = \text{dot}(n_e, 0.5 - v_{proj}) + 0.5 \times \max(|n_{e,x}|, |n_{e,y}|) $$
   For **Fat** voxelization [S3]:
   $$ d_e = -\text{dot}(n_e, v_{proj}) + \max(0, n_{e,x}) + \max(0, n_{e,y}) $$
   Compute triangle plane distance [S3]:
   Thin: $d_{Tri} = \text{dot}(n_{proj}, v_0) - \text{dot}(n_{proj,xy}, 0.5)$
   Fat: $d_{Tri, min} = \text{dot}(n_{proj}, v_0) - \max(n_{proj,x}, 0) - \max(n_{proj,y}, 0)$
   $d_{Tri, max} = \text{dot}(n_{proj}, v_0) - \min(n_{proj,x}, 0) - \min(n_{proj,y}, 0)$
   [S3].

6. **X-Y Loop (Rasterization):**
   Compute AABB of triangle vertices. Convert to voxel indices:
   $$ minVoxIndex = \text{clamp}(\lfloor AABB_{min} \rfloor, 0, voxelResolution) $$
   $$ maxVoxIndex = \text{clamp}(\lceil AABB_{max} \rceil, 0, voxelResolution) $$
   [S3].
   Iterate $p.x$ from $minVoxIndex.x$ to $maxVoxIndex.x$ and $p.y$ from $minVoxIndex.y$ to $maxVoxIndex.y$ [S3].
   Calculate updated edge distances for current voxel coordinate:
   $$ dd_e = d_e + \text{dot}(n_e, p_{proj}) $$
   [S3].
   Check XY overlap:
   $$ xy\_overlap = (dd_{e0,xy} \ge 0) \land (dd_{e1,xy} \ge 0) \land (dd_{e2,xy} \ge 0) $$
   [S3].

7. **Z-Range Calculation:**
   If $xy\_overlap$ is true, calculate Z-intersection range [S3].
   Thin [S3]:
   $$ z_{min} = \frac{-\text{dot}(n_{proj,xy}, p_{xy}) + d_{Tri, thin}}{n_{proj, z}} $$
   Fat [S3]:
   $$ z_{min} = \frac{-\text{dot}(n_{proj,xy}, p_{xy}) + d_{Tri, fat, min}}{n_{proj, z}} $$
   $$ z_{max} = \frac{-\text{dot}(n_{proj,xy}, p_{xy}) + d_{Tri, fat, max}}{n_{proj, z}} $$
   Apply floor/ceil boundary handling:
   $$ zMin = \text{int}(zMinFloor) - \text{int}(zMinFloor == zMinInt) $$
   $$ zMax = \text{int}(zMaxCeil) + \text{int}(zMaxCeil == zMaxInt) $$
   [S3].

8. **Z-Loop and Overlap Check:**
   Iterate $p.z$ from $zMin$ to $zMax$ (clamped to $minVoxIndex.z$ and $maxVoxIndex.z$) [S3].
   Inside loop, calculate updated edge distances for YZ and ZX planes:
   $$ dd_e = d_e + \text{dot}(n_e, p_{proj}) $$
   [S3].
   Check overlaps:
   $$ yz\_overlap = (dd_{e0,yz} \ge 0) \land \dots $$
   $$ zx\_overlap = (dd_{e0,zx} \ge 0) \land \dots $$
   [S3].

9. **Write Voxel:**
   If $yz\_overlap$ and $zx\_overlap$ are true, the voxel is occupied [S3].
   Write to occupancy image:
   $$ \text{imageStore}(voxelOccupancy, \text{ivec3}(unswizzle \cdot p), \text{uvec4}(1)) $$
   Write color:
   $$ \text{imageStore}(voxelColor, \text{ivec3}(unswizzle \cdot p), \text{vec4}(unswizzle \cdot p / voxelResolution, 1)) $$
   [S3].

10. **Synchronize and Read Back:**
    Call `glMemoryBarrier(GL_ALL_BARRIER_BITS)` to ensure 3D texture write is complete [S3]. Retrieve 3D texture data using `glGetTexImage` into `std::vector<uint8_t>` [S3]. Iterate through grid in order z (i), y (j), x (k). If `image[index]` is non-zero, compute normalized position $(k/res, j/res, i/res)$ and store in `position` vector [S3].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def geometry_shader_voxelization(mesh_V, mesh_F, voxel_resolution, thickness_mode):
    """
    mesh_V: 3xN float array of vertices
    mesh_F: 3xM uint32 array of face indices
    voxel_resolution: int, dimension of cubic grid
    thickness_mode: 'THIN' or 'FAT'
    """
    # 1. Normalize Mesh
    min_coords = np.min(mesh_V, axis=1)
    max_coords = np.max(mesh_V, axis=1)
    dmax = np.max(max_coords - min_coords)
    mesh_V_norm = (mesh_V - min_coords) / dmax
    
    # 2. Setup Pipeline (Abstracted)
    # Bind shaders, buffers, and images (voxelOccupancy, voxelColor)
    
    # 3. Process Each Triangle
    for i in range(mesh_F.shape[1]):
        v0 = mesh_V_norm[:, mesh_F[0, i]]
        v1 = mesh_V_norm[:, mesh_F[1, i]]
        v2 = mesh_V_norm[:, mesh_F[2, i]]
        
        # Swizzle: Align dominant normal axis to Z (XY plane projection)
        n = np.cross(v1 - v0, v2 - v1)
        dominant_axis = np.argmax(np.abs(n))
        # Permute vertices and normal to align dominant plane with XY
        v0_swizzled, v1_swizzled, v2_swizzled, n_swizzled, unswizzle = swizzle_triangle(v0, v1, v2, n)
        
        # 4. Compute Edge Normals
        e0 = v1_swizzled - v0_swizzled
        e1 = v2_swizzled - v1_swizzled
        e2 = v0_swizzled - v2_swizzled
        
        # Compute 2D edge normals for XY, YZ, ZX planes
        # Example for XY plane (using n.z)
        n_e0_xy = np.array([-e0[1], e0[0]]) if n_swizzled[2] >= 0 else np.array([e0[1], -e0[0]])
        n_e1_xy = np.array([-e1[1], e1[0]]) if n_swizzled[2] >= 0 else np.array([e1[1], -e1[0]])
        n_e2_xy = np.array([-e2[1], e2[0]]) if n_swizzled[2] >= 0 else np.array([e2[1], -e2[0]])
        
        # Similar logic for YZ (using n.x) and ZX (using n.y) planes...
        
        # 5. Calculate Initial Distances
        # Projected vertices for XY plane
        v0_proj = v0_swizzled[:2]
        v1_proj = v1_swizzled[:2]
        v2_proj = v2_swizzled[:2]
        
        if thickness_mode == 'THIN':
            d_e0_xy = dot(n_e0_xy, 0.5 - v0_proj) + 0.5 * max(abs(n_e0_xy[0]), abs(n_e0_xy[1]))
            # ... compute d_e1_xy, d_e2_xy
            d_tri = dot(n_swizzled[:2], v0_proj) - dot(n_swizzled[:2], 0.5)
        else: # FAT
            d_e0_xy = -dot(n_e0_xy, v0_proj) + max(0, n_e0_xy[0]) + max(0, n_e0_xy[1])
            # ... compute d_e1_xy, d_e2_xy
            d_tri_min = dot(n_swizzled[:2], v0_proj) - max(n_swizzled[0], 0) - max(n_swizzled[1], 0)
            d_tri_max = dot(n_swizzled[:2], v0_proj) - min(n_swizzled[0], 0) - min(n_swizzled[1], 0)
            
        # 6. X-Y Loop
        # Compute AABB in voxel space
        min_vox = clamp(floor(min(v0_proj, v1_proj, v2_proj)), 0, voxel_resolution)
        max_vox = clamp(ceil(max(v0_proj, v1_proj, v2_proj)), 0, voxel_resolution)
        
        for p_x in range(min_vox[0], max_vox[0] + 1):
            for p_y in range(min_vox[1], max_vox[1] + 1):
                p_proj = np.array([p_x, p_y])
                
                # Update distances
                dd_e0_xy = d_e0_xy + dot(n_e0_xy, p_proj)
                dd_e1_xy = d_e1_xy + dot(n_e1_xy, p_proj)
                dd_e2_xy = d_e2_xy + dot(n_e2_xy, p_proj)
                
                if (dd_e0_xy >= 0) and (dd_e1_xy >= 0) and (dd_e2_xy >= 0):
                    # 7. Z-Range Calculation
                    if thickness_mode == 'THIN':
                        z_min = (-dot(n_swizzled[:2], p_proj) + d_tri) / n_swizzled[2]
                        z_max = z_min # Simplified for thin, actual logic involves floor/ceil boundary handling
                    else:
                        z_min = (-dot(n_swizzled[:2], p_proj) + d_tri_min) / n_swizzled[2]
                        z_max = (-dot(n_swizzled[:2], p_proj) + d_tri_max) / n_swizzled[2]
                    
                    # Apply boundary handling
                    z_min_int = int(z_min)
                    z_max_int = int(z_max)
                    z_min_final = z_min_int - int(z_min_int == z_min)
                    z_max_final = z_max_int + int(z_max_int == z_max)
                    
                    # Clamp Z range to grid
                    z_min_final = max(0, min(z_min_final, voxel_resolution))
                    z_max_final = max(0, min(z_max_final, voxel_resolution))
                    
                    # 8. Z-Loop and Overlap Check
                    for p_z in range(z_min_final, z_max_final + 1):
                        p_swizzled = np.array([p_x, p_y, p_z])
                        
                        # Check YZ and ZX overlaps
                        # ... (similar distance updates and checks for YZ and ZX planes)
                        if yz_overlap and zx_overlap:
                            # 9. Write Voxel
                            p_unswizzled = unswizzle @ p_swizzled
                            image_store(voxelOccupancy, p_unswizzled, 1)
                            image_store(voxelColor, p_unswizzled, p_unswizzled / voxel_resolution)

    # 10. Synchronize
    gl_memory_barrier(GL_ALL_BARRIER_BITS)
    return read_3d_texture(voxelOccupancy)
```

## Subtopics

### [GPU Mesh Voxelization via AABB-Triangle Intersection](2541-gpu-mesh-voxelization-via-aabb-triangle-intersection.md)
A GPU-based method to convert a triangle mesh into a set of voxel points by dispatching one compute thread per grid cell and testing each cell's Axis-Aligned Bounding Box (AABB) against all mesh triangles.

### [GPU Voxelization Using OpenGL Image Atomics](2544-gpu-voxelization-using-opengl-image-atomics.md)
A technique that transforms continuous 3D polygonal meshes into a discrete, regular volumetric grid stored in GPU memory by projecting geometry along its dominant axis, rasterizing fragments without a traditional framebuffer, and utilizing fragment-stage image atomic operations to write directly into 3D textures without concurrency conflicts.

### [GPU Mesh Voxelization via Axis-Aligned Projection](2561-gpu-mesh-voxelization-via-axis-aligned-projection.md)
GPU Mesh Voxelization via Axis-Aligned Projection is a rasterization-based technique that converts polygonal triangle meshes into a 3D voxel grid by projecting each triangle onto a 2D plane aligned with its dominant normal axis. It solves the problem of efficiently filling a 3D occupancy or color grid using standard GPU hardware rasterizers, which are optimized for 2D screens. Unlike [Geometry Shader Voxelization](2566-geometry-shader-voxelization.md) which may use generic 3D rasterization logic, this method explicitly selects the optimal projection axis per triangle to maximize fragment coverage and minimize holes. It differs from [GPU Mesh Voxelization via AABB-Triangle Intersection](2541-gpu-mesh-voxelization-via-aabb-triangle-intersection.md) by relying on hardware rasterization rather than analytical intersection tests, and from [GPU Voxelization Using OpenGL Image Atomics](2544-gpu-voxelization-using-opengl-image-atomics.md) by typically using a single draw call with axis-specific view matrices rather than multiple passes or pure atomic accumulation without depth-based projection.

## Operating characteristics
- **Time Complexity:** Depends on the number of triangles and the volume of their bounding boxes in voxel space. The nested loops (XY and Z) iterate over the bounding box of each triangle, making it efficient for sparse scenes but potentially costly for dense or large triangles.
- **Memory Complexity:** $O(N^3)$ for a grid of size $N \times N \times N$ to store the occupancy and color data [S3].
- **Performance:** Efficient for real-time voxelization on GPUs supporting Geometry Shaders [S3]. Performance scales with voxel resolution and triangle count.
- **Strengths:** Uses standard graphics pipeline, easy to implement with OpenGL, supports both thin and fat voxelization [S3].
- **Limitations:** Geometry shader overhead, potential for holes if not using fat voxelization, requires careful handling of boundary conditions [S3].
- **When to use:** When using OpenGL and needing a straightforward rasterization-based approach.
- **When not to use:** When using DirectX 12 without geometry shaders (use [VoxelizerX](2566-geometry-shader-voxelization.md) DX12 variant or other methods), or when compute shader-based methods are preferred for better control.

## Background
Originated from OpenGL geometry shader capabilities, allowing emission of multiple primitives from a single input primitive [S3]. Used in lightweight mesh voxelization libraries for real-time applications [S3].

## Sources
[S1] ivandrodri/gpu-voxelizer, https://github.com/ivandrodri/gpu-voxelizer, fetched 2026-10-26 (unofficial)
[S2] StarsX/VoxelizerX12, https://github.com/StarsX/VoxelizerX12, fetched 2026-10-27 (unofficial)
[S3] skanti/Mesh-Voxelization-OpenGL-Geometry-Shader, https://github.com/skanti/Mesh-Voxelization-OpenGL-Geometry-Shader, fetched 2026-10-27 (unofficial)
