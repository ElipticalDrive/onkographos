# Voxelization by Rasterization

Also called: Hybrid Ambient Occlusion (HAO), Occupancy grid generation via rasterization, Voxelization by Eisemann and Décoret

## Overview
This method generates a 3D occupancy grid (voxel proxy) by rasterizing scene geometry into a 3D texture using a single-pass orthographic projection. It solves the problem of capturing geometry outside the view frustum or occluded by front-facing objects for downstream applications like Hybrid Ambient Occlusion (HAO) [S1]. It differs from standard screen-space techniques by operating in object space, allowing it to consider occluders that are not currently visible on the screen [S1].

## How it works
**Inputs:**
- Scene geometry (triangles) within a defined bounding box.
- Orthographic camera parameters defining the voxelization volume (near/far planes).
- Target 3D resolution ($N_x, N_y, N_z$).

**Outputs:**
- A 3D texture representing the occupancy grid.

**Data Structures:**
- **Voxel Grid:** A 3D texture where each slice represents a layer along the depth axis.
- **Cell Representation:** Each pixel in the 3D texture holds one to four 32-bit unsigned integer values. A voxel cell is described by a single bit indicating the presence of geometry in its associated sub-volume [S1].
- **Bit Packing:** Up to 8 output buffers can be used simultaneously to achieve higher resolutions. The maximum 3D resolution ranges from $256^3$ to $1024^3$ depending on the pixel format and number of output buffers [S1].

**Steps:**
1. **Setup Voxelization Camera:** Define an orthographic camera that encompasses the region of interest (bounding box).
2. **Disable Pipeline Operations:** Disable texturing, depth testing, and other unnecessary operations to optimize performance [S1].
3. **Rasterize Geometry:** Render the scene geometry from the orthographic camera.
4. **Logical Combination:** Combine newly rasterized fragments with existing data in the framebuffer using a logical OR operator. This ensures that if any part of a triangle covers a voxel cell, that cell is marked as occupied [S1].
5. **Handle Axis-Aligned Artifacts:** To prevent polygons aligned with the viewing direction from having a zero framebuffer footprint (and thus being missed), rotate the voxelizing camera by a random angle prior to voxelization [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxelize_by_rasterization(geometry, bounding_box, resolution, num_buffers):
    """
    Generates a 3D occupancy grid by rasterizing geometry.
    
    Args:
        geometry: List of triangles (vertices).
        bounding_box: AABB defining the voxelization volume.
        resolution: Tuple (Nx, Ny, Nz) for the 3D texture dimensions.
        num_buffers: Number of color buffers to use for bit packing (1-8).
        
    Returns:
        3D texture (Nx, Ny, Nz) with bits set for occupied cells.
    """
    # Initialize 3D texture with zeros
    voxel_grid = initialize_3d_texture(resolution, num_buffers)
    
    # Create orthographic camera for the bounding box
    camera = create_orthographic_camera(bounding_box)
    
    # Rotate camera by a random angle to avoid axis-aligned artifacts
    # [G] Random rotation prevents zero-area projection of axis-aligned polygons
    camera.rotate(random_rotation())
    
    # Disable depth test and texturing
    set_pipeline_state(depth_test=False, texturing=False)
    
    # Set up render targets for the 3D texture slices
    # Each slice corresponds to a layer in the 3D texture
    set_render_targets(voxel_grid)
    
    # Clear the texture to 0
    clear_render_targets(0)
    
    # Render geometry
    # The rasterizer projects triangles onto the 3D texture slices
    # Fragments are combined using logical OR
    for triangle in geometry:
        if intersects_bounding_box(triangle, bounding_box):
            rasterize_triangle(triangle, camera, voxel_grid, operation=LOGICAL_OR)
            
    return voxel_grid
```

**Parameters:**
- **Resolution:** Maximum 3D resolution ranges from $256^3$ to $1024^3$ depending on the pixel format and number of output buffers (up to 8) [S1].
- **Bit Depth:** Each voxel cell is described by a single bit [S1].

**Edge Cases and Pitfalls:**
- **Axis-Aligned Artifacts:** Polygons aligned with the viewing direction of the voxelizing camera have a framebuffer footprint of zero and are not stored. Solution: Rotate the voxelizing camera by a random angle prior to voxelization [S1].
- **Precision:** Numerical precision issues may cause axis-aligned polygons to be missed if the camera is not rotated [S1].

## Subtopics

### [3-axis rasterization voxelization](2480-3-axis-rasterization-voxelization.md)
Uses three separate orthographic passes along the X, Y, and Z axes. Chosen when higher fidelity is needed at the cost of performance.

### [Chunked UV Rasterization with Face ID Storage](2598-chunked-uv-rasterization-with-face-id-storage.md)
Stores face IDs in addition to occupancy. Chosen when material or normal information per voxel is required.

### [Triangle Projection Voxelization](5432-triangle-projection-voxelization.md)
Projects triangles directly into voxels without rasterization. Chosen for precise boundary representation.

### [DX11.1 Forced Sample Count Optimization](8063-dx11-1-forced-sample-count-optimization.md)
Uses DX11.1 features to optimize sample counts. Chosen for specific hardware optimizations.

### [Bone-Animated Voxel Skinning with Skeletal Mesh Voxelization](12565-bone-animated-voxel-skinning-with-skeletal-mesh-voxelization.md)
Handles skeletal animation during voxelization. Chosen for animated characters.

### [Cascaded Voxel Cone Tracing Voxelization](13853-cascaded-voxel-cone-tracing-voxelization.md)
Uses cascaded volumes for global illumination. Chosen for advanced lighting effects.

## Operating characteristics
- **Time Complexity:** Single-pass rasterization makes it fast, suitable for real-time applications [S1].
- **Memory Complexity:** Depends on resolution and bit packing. Up to 8 buffers allow higher resolution but increase memory usage [S1].
- **Strengths:** Real-time performance, handles dynamic scenes without CPU precomputation, considers geometry outside the view frustum [S1].
- **Limitations:** Axis-aligned artifacts if camera is not rotated, lower fidelity than ray-traced voxelization [S1].
- **When to Use:** Real-time ambient occlusion in dynamic scenes where objects outside the view frustum contribute to occlusion [S1].
- **When Not to Use:** When precise geometric boundaries are required, or when hardware does not support multiple render targets efficiently [S1].

## Background
The method was introduced by Eisemann and Décoret in 2006 as part of the Hybrid Ambient Occlusion (HAO) technique [S1]. It was designed to overcome the limitations of screen-space ambient occlusion (SSAO) by incorporating object-space information. The technique has been used in real-time rendering engines for dynamic scenes [S1].

## Sources
[S1] Hybrid Ambient Occlusion, https://doi.org/10.2312/ega.20091008.051-057, fetched 2026-11-03
