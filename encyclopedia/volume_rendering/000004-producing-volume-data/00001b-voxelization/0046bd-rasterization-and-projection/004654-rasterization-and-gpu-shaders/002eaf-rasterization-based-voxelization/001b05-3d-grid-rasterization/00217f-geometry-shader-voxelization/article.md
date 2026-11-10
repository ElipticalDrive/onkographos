# Geometry Shader Voxelization

Also called: GS Voxelization, Solid Triangle Voxelization, Triangle-to-Voxel Rasterization, OpenGL Geometry Shader Voxelization

## Overview
Geometry Shader Voxelization converts polygonal meshes (`.obj`, `.ply`) into a discrete 3D occupancy grid by rasterizing triangles directly into a 3D texture using an OpenGL Geometry Shader [S3, unofficial]. It determines occupied voxels by projecting triangles onto axis-aligned planes, computing edge distances, and iterating through the triangle's bounding box to check for overlaps in all three dimensions (XY, YZ, ZX) [S3, unofficial]. Unlike [A-Buffer Solid Voxelization](1189-a-buffer-solid-voxelization.md) which uses a 2D accumulation buffer to resolve depth, this method explicitly tests 3D spatial overlap per voxel index within the shader loop, ensuring watertightness without traditional framebuffer depth buffers [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangle mesh defined by vertex positions $V$ (size $3 \times N$) and face indices $F$ (size $3 \times M$) [S3, unofficial].
*   **Output:** A 3D occupancy grid (texture) where occupied voxels are marked with a flag (e.g., 1) and optionally color data [S3, unofficial].

**Data Structures:**
*   **Voxel Resolution:** A cubic grid dimension `dim` (e.g., 64), applied uniformly to x, y, z axes [S3, unofficial].
*   **Occupancy Grid:** A 3D texture (`GL_TEXTURE_3D`) with internal format `GL_R8UI` (8-bit unsigned integer, single channel) [S3, unofficial]. Dimensions are `voxelResolution[0] x voxelResolution[1] x voxelResolution[2]` [S3, unofficial]. Bound as an image buffer `voxelOccupancy` at binding point 0 with access `GL_READ_WRITE` [S3, unofficial].
*   **Color Buffer (Optional):** A 3D image `voxelColor` with format `rgba8` storing color data [S3, unofficial].
*   **Unswizzle LUT:** A lookup table storing permutation matrices to reverse coordinate transformations applied during dominant axis alignment [S3, unofficial].

**Parameters:**
*   `voxelResolution`: Uniform `ivec3` specifying grid dimensions [S3, unofficial].
*   `THICKNESS`: Compile-time constant (`0` for Thin, `1` for Fat) selecting the voxelization style [S3, unofficial].

**Steps:**

1.  **Load and Normalize Mesh:**
    Load vertices $V$ and faces $F$ from `.obj` or `.ply` files [S3, unofficial]. Compute the bounding box of vertices: $xmin, xmax, ymin, ymax, zmin, zmax$ [S3, unofficial]. Calculate dimensions $dx, dy, dz$ and $dmax = \max(dx, dy, dz)$ [S3, unofficial]. Normalize vertex coordinates to the $[0, 1]$ range: $V_{norm} = (V - min) / dmax$ [S3, unofficial].

2.  **Setup Rendering Pipeline:**
    Load shaders (`VoxelizationVS.glsl`, `VoxelizationGS.glsl`, `VoxelizationFS.glsl`) [S3, unofficial]. Bind vertex buffer $V$ and element buffer $F$ [S3, unofficial]. Pass `voxelResolution` uniform [S3, unofficial].

3.  **Swizzle Triangle (Dominant Axis Alignment):**
    For each triangle, compute the normal $n = \text{cross}(v_1 - v_0, v_2 - v_1)$ [S3, unofficial]. Determine the dominant axis of the normal (the component with the maximum absolute value) [S3, unofficial]. Permute vertex coordinates and normal to align the dominant plane with the XY plane [S3, unofficial]. Store the permutation matrix in `unswizzle` from `unswizzleLUT` to reverse the transformation later [S3, unofficial].

4.  **Compute Edge Normals:**
    For each edge ($e_0=v_1-v_0, e_1=v_2-v_1, e_2=v_0-v_2$), compute inward-facing 2D edge normals for the XY, YZ, and ZX planes [S3, unofficial]. The direction depends on the sign of the normal component perpendicular to the plane [S3, unofficial].
    *   For the XY plane (perpendicular to Z):
        $$n_{e0,xy} = \begin{cases} (-e_{0,y}, e_{0,x}) & \text{if } n_z \ge 0 \\ (e_{0,y}, -e_{0,x}) & \text{if } n_z < 0 \end{cases}$$
    *   Similar logic applies for YZ (perpendicular to X) and ZX (perpendicular to Y) planes [S3, unofficial].

5.  **Calculate Initial Edge Distances:**
    Compute the signed distance from the origin to the triangle plane along the projected normal [S3, unofficial].
    *   **Thin Voxelization:**
        $$d_e = \text{dot}(n_e, 0.5 - v_{proj}) + 0.5 \times \max(|n_{e,x}|, |n_{e,y}|)$$
        $$d_{Tri} = \text{dot}(n_{proj}, v_0) - \text{dot}(n_{proj,xy}, 0.5)$$
    *   **Fat Voxelization:**
        $$d_e = -\text{dot}(n_e, v_{proj}) + \max(0, n_{e,x}) + \max(0, n_{e,y})$$
        $$d_{Tri, min} = \text{dot}(n_{proj}, v_0) - \max(n_{proj,x}, 0) - \max(n_{proj,y}, 0)$$
        $$d_{Tri, max} = \text{dot}(n_{proj}, v_0) - \min(n_{proj,x}, 0) - \min(n_{proj,y}, 0)$$
    These are computed for all 3 edges and 3 planes (XY, YZ, ZX) [S3, unofficial].

6.  **Bounding Box and Loop Bounds:**
    Compute the axis-aligned bounding box (AABB) of the triangle vertices in the normalized space [S3, unofficial]. Convert to voxel indices:
    $$minVoxIndex = \text{clamp}(\lfloor AABB_{min} \rfloor, 0, voxelResolution)$$
    $$maxVoxIndex = \text{clamp}(\lceil AABB_{max} \rceil, 0, voxelResolution)$$
    [S3, unofficial].

7.  **Iterate and Test Overlap:**
    *   **X-Y Loop:** Iterate $p.x$ from $minVoxIndex.x$ to $maxVoxIndex.x$ and $p.y$ from $minVoxIndex.y$ to $maxVoxIndex.y$ [S3, unofficial].
        Calculate updated edge distances for the current voxel coordinate:
        $$dd_e = d_e + \text{dot}(n_e, p_{proj})$$
        Check if the voxel overlaps the triangle in the XY plane:
        $$xy\_overlap = (dd_{e0,xy} \ge 0) \land (dd_{e1,xy} \ge 0) \land (dd_{e2,xy} \ge 0)$$
        [S3, unofficial].
    *   **Z-Range Calculation:** If $xy\_overlap$ is true, calculate the Z-intersection range [S3, unofficial].
        *   **Thin:**
            $$z_{min} = \frac{-\text{dot}(n_{proj,xy}, p_{xy}) + d_{Tri, thin}}{n_{proj, z}}$$
            Boundary handling: $zMin = \lfloor z_{min} \rfloor - \mathbb{I}(\lfloor z_{min} \rfloor == z_{min})$ and $zMax = \lceil z_{max} \rceil + \mathbb{I}(\lceil z_{max} \rceil == z_{max})$ [S3, unofficial].
        *   **Fat:**
            $$z_{min} = \frac{-\text{dot}(n_{proj,xy}, p_{xy}) + d_{Tri, fat, min}}{n_{proj, z}}, \quad z_{max} = \frac{-\text{dot}(n_{proj,xy}, p_{xy}) + d_{Tri, fat, max}}{n_{proj, z}}$$
            Apply similar floor/ceil boundary handling [S3, unofficial].
    *   **Z-Loop:** Iterate $p.z$ from $zMin$ to $zMax$ (clamped to $minVoxIndex.z$ and $maxVoxIndex.z$) [S3, unofficial].
        Inside the loop, calculate updated edge distances for YZ and ZX planes:
        $$dd_e = d_e + \text{dot}(n_e, p_{proj})$$
        Check overlaps:
        $$yz\_overlap = (dd_{e0,yz} \ge 0) \land (dd_{e1,yz} \ge 0) \land (dd_{e2,yz} \ge 0)$$
        $$zx\_overlap = (dd_{e0,zx} \ge 0) \land (dd_{e1,zx} \ge 0) \land (dd_{e2,zx} \ge 0)$$
        [S3, unofficial].

8.  **Write Voxel:**
    If $yz\_overlap$ and $zx\_overlap$ are true, the voxel is occupied [S3, unofficial].
    Write to the occupancy image:
    $$\text{imageStore}(voxelOccupancy, \text{ivec3}(unswizzle \cdot p), \text{uvec4}(1))$$
    Write color (optional):
    $$\text{imageStore}(voxelColor, \text{ivec3}(unswizzle \cdot p), \text{vec4}(unswizzle \cdot p / voxelResolution, 1))$$
    [S3, unofficial].

9.  **Synchronize and Read Back:**
    Call `glMemoryBarrier(GL_ALL_BARRIER_BITS)` to ensure the 3D texture write is complete [S3, unofficial]. Retrieve the 3D texture data into a `std::vector<uint8_t>` using `glGetTexImage` [S3, unofficial]. Iterate through the grid in order z (i), y (j), x (k) [S3, unofficial]. If `image[index]` is non-zero, compute normalized position $(k/res, j/res, i/res)$ and store in `position` vector [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def geometry_shader_voxelization(mesh_V, mesh_F, voxel_resolution):
    """
    Voxelize a mesh using the Geometry Shader method.
    mesh_V: Nx3 array of vertex positions.
    mesh_F: Mx3 array of vertex indices.
    voxel_resolution: int, dimension of the cubic grid.
    """
    import numpy as np
    
    # 1. Normalize Mesh
    min_coords = mesh_V.min(axis=0)
    max_coords = mesh_V.max(axis=0)
    dmax = max(max_coords - min_coords)
    V_norm = (mesh_V - min_coords) / dmax
    
    # Initialize occupancy grid
    occupancy_grid = np.zeros((voxel_resolution, voxel_resolution, voxel_resolution), dtype=np.uint8)
    
    # 3. Process Each Triangle
    for i in range(mesh_F.shape[0]):
        v0 = V_norm[mesh_F[i, 0]]
        v1 = V_norm[mesh_F[i, 1]]
        v2 = V_norm[mesh_F[i, 2]]
        
        # 4. Swizzle (Dominant Axis)
        n = np.cross(v1 - v0, v2 - v1)
        dominant_axis = np.argmax(np.abs(n))
        # Permute v0, v1, v2 and n to align dominant axis with Z (for XY projection)
        # Store unswizzle permutation matrix P_unswizzle
        unswizzle = np.eye(3) # In practice, permute based on dominant_axis
        
        # 5. Compute Edge Normals and Distances
        e0 = v1 - v0
        e1 = v2 - v1
        e2 = v0 - v2
        edges = [e0, e1, e2]
        
        # Compute 2D edge normals for XY, YZ, ZX planes
        # Example for XY plane (perp to Z):
        n_e0_xy = np.array([-e0[1], e0[0]]) if n[2] >= 0 else np.array([e0[1], -e0[0]])
        n_e1_xy = np.array([-e1[1], e1[0]]) if n[2] >= 0 else np.array([e1[1], -e1[0]])
        n_e2_xy = np.array([-e2[1], e2[0]]) if n[2] >= 0 else np.array([e2[1], -e2[0]])
        
        # Compute initial distances based on THICKNESS (Thin/Fat)
        # Using Thin voxelization as default
        d_e0_xy = np.dot(n_e0_xy, 0.5 - v0[:2]) + 0.5 * max(abs(n_e0_xy[0]), abs(n_e0_xy[1]))
        d_e1_xy = np.dot(n_e1_xy, 0.5 - v1[:2]) + 0.5 * max(abs(n_e1_xy[0]), abs(n_e1_xy[1]))
        d_e2_xy = np.dot(n_e2_xy, 0.5 - v2[:2]) + 0.5 * max(abs(n_e2_xy[0]), abs(n_e2_xy[1]))
        d_Tri = np.dot(n[:2], v0[:2]) - np.dot(n[:2], 0.5)
        
        # 6. Bounding Box
        aabb_min = np.minimum(np.minimum(v0, v1), v2)
        aabb_max = np.maximum(np.maximum(v0, v1), v2)
        min_vox = np.clip(np.floor(aabb_min).astype(int), 0, voxel_resolution - 1)
        max_vox = np.clip(np.ceil(aabb_max).astype(int), 0, voxel_resolution - 1)
        
        # 7. Iterate and Test
        for px in range(min_vox[0], max_vox[0] + 1):
            for py in range(min_vox[1], max_vox[1] + 1):
                # Check XY overlap
                dd_e0_xy = d_e0_xy + np.dot(n_e0_xy, [px, py])
                dd_e1_xy = d_e1_xy + np.dot(n_e1_xy, [px, py])
                dd_e2_xy = d_e2_xy + np.dot(n_e2_xy, [px, py])
                xy_overlap = (dd_e0_xy >= 0) and (dd_e1_xy >= 0) and (dd_e2_xy >= 0)
                
                if xy_overlap:
                    # Calculate Z range
                    z_min_calc = (-np.dot(n[:2], [px, py]) + d_Tri) / n[2]
                    z_max_calc = z_min_calc # Simplified; actual fat/thin logic differs
                    z_min = np.clip(np.floor(z_min_calc).astype(int), 0, voxel_resolution - 1)
                    z_max = np.clip(np.ceil(z_max_calc).astype(int), 0, voxel_resolution - 1)
                    
                    for pz in range(z_min, z_max + 1):
                        # Check YZ and ZX overlaps
                        # (Simplified for brevity; follows same dot product logic as XY)
                        yz_overlap = True
                        zx_overlap = True
                        
                        if yz_overlap and zx_overlap:
                            # 8. Write Voxel
                            idx = unswizzle @ np.array([px, py, pz])
                            occupancy_grid[idx[0], idx[1], idx[2]] = 1
                            
    # 9. Synchronize and Read Back
    # glMemoryBarrier, glGetTexImage
    return occupancy_grid
```

## Subtopics

### [GPU Mesh Voxelization via AABB-Triangle Intersection](2541-gpu-mesh-voxelization-via-aabb-triangle-intersection.md)
A GPU-based brute-force method to convert a triangle mesh into a set of voxel points by dispatching one compute thread per grid cell and testing each cell's Axis-Aligned Bounding Box (AABB) against all mesh triangles. It differs from [Geometry Shader Voxelization](2566-geometry-shader-voxelization.md) by using a compute shader with explicit AABB-triangle intersection tests rather than rasterization, and from [GPU Mesh Voxelization via Axis-Aligned Projection](2561-gpu-mesh-voxelization-via-axis-aligned-projection.md) by iterating all triangles per voxel rather than projecting triangles into grid slices.

### [GPU Voxelization Using OpenGL Image Atomics](2544-gpu-voxelization-using-opengl-image-atomics.md)
GPU Voxelization Using OpenGL Image Atomics transforms continuous 3D polygonal meshes into a discrete, regular volumetric grid stored in GPU memory by projecting geometry along its dominant axis, rasterizing fragments without a traditional framebuffer, and utilizing fragment-stage image atomic operations to write directly into 3D textures. It solves the problems of projection loss (degenerate triangles perpendicular to the view plane) and race conditions (concurrent writes to the same voxel) inherent in standard rasterization pipelines. Unlike standard rasterization which projects onto a 2D viewport, this method decouples rasterization from the framebuffer, converting complex geometric meshes into solid or shell voxel structures in a single render pass. It differs from [Geometry Shader Voxelization](2566-geometry-shader-voxelization.md) by explicitly handling concurrency via hardware atomics and dominant-axis projection to ensure watertightness, and from [GPU Mesh Voxelization via AABB-Triangle Intersection](2541-gpu-mesh-voxelization-via-aabb-triangle-intersection.md) by relying on rasterization primitives rather than explicit geometric intersection tests.

### [GPU Mesh Voxelization via Axis-Aligned Projection](2561-gpu-mesh-voxelization-via-axis-aligned-projection.md)
GPU Mesh Voxelization via Axis-Aligned Projection converts triangle meshes into a 3D voxel grid by rasterizing each triangle orthographically along its dominant normal axis. It solves the "missing voxel" artifact inherent in single-direction rasterization, where surfaces viewed edge-on leave gaps in the 3D texture. Unlike [GPU Voxelization Using OpenGL Image Atomics](2544-gpu-voxelization-using-opengl-image-atomics.md) which may require multiple passes or complex atomic logic for depth, and [GPU Mesh Voxelization via AABB-Triangle Intersection](2541-gpu-mesh-voxelization-via-aabb-triangle-intersection.md) which relies on geometric intersection tests, this method uses the GPU's native rasterizer to fill fragments, ensuring maximal coverage per triangle with a single pass.

## Operating characteristics
*   **Time Complexity:** Proportional to the number of triangles multiplied by the volume of their bounding boxes in voxel space [S3, unofficial]. The nested loops (XY and Z) make it sensitive to triangle size and orientation [S3, unofficial].
*   **Memory Complexity:** $O(R^3)$ for the 3D occupancy grid, where $R$ is the voxel resolution [S3, unofficial].
*   **Strengths:** Uses standard graphics pipeline primitives (Geometry Shader) for explicit 3D overlap testing, ensuring watertightness without depth buffer artifacts [S3, unofficial]. Supports both "Thin" (vertex-connected) and "Fat" (face-connected) voxelization [S3, unofficial].
*   **Limitations:** Requires explicit iteration through bounding boxes in the shader, which can be slower than simple rasterization for large, sparse scenes [S3, unofficial]. Boundary clamping is critical to prevent out-of-bounds access [S3, unofficial].
*   **When to use:** When watertight voxelization is required and the scene geometry is not excessively large or sparse [G]. Suitable for OpenGL-based pipelines [S3, unofficial].
*   **When not to use:** For very large, sparse scenes where sparse voxel structures (like [Adaptive Sparse Voxelization with Bounding Box Thickness Cases](1686-adaptive-sparse-voxelization-with-bounding-box-thickness-cas.md)) are more memory-efficient [G].

## Background
Geometry Shader Voxelization is a method for voxelizing polygonal meshes using OpenGL Geometry Shaders [S3, unofficial]. It was implemented in the open-source library `Mesh-Voxelization-OpenGL-Geometry-Shader` by skanti [S3, unofficial]. The method leverages the geometry shader's ability to generate new primitives (in this case, iterating through voxel indices) to perform explicit 3D rasterization [S3, unofficial].

## Sources
[S1] ivandrodri/gpu-voxelizer, https://github.com/ivandrodri/gpu-voxelizer, fetched 2026-10-26 (unofficial)
[S2] StarsX/VoxelizerX12, https://github.com/StarsX/VoxelizerX12, fetched 2026-10-27 (unofficial)
[S3] skanti/Mesh-Voxelization-OpenGL-Geometry-Shader, https://github.com/skanti/Mesh-Voxelization-OpenGL-Geometry-Shader, fetched 2026-10-27 (unofficial)
