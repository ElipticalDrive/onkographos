# MSAA Voxelization

Also called: 8xMSAA Voxelization, Conservative Raster via MSAA, Conservative Rasterization with MSAA render target, Multi-Sampling Anti-Aliasing Voxelization

## Overview
MSAA Voxelization is a GPU-based technique that approximates conservative rasterization by leveraging Multi-Sampling Anti-Aliasing (MSAA) coverage tests to determine voxel inclusion, thereby avoiding the computational overhead of explicit primitive edge expansion in the Geometry Shader [S1, unofficial]. It differs from sibling methods like [Per-Pixel Linked List Voxelization](1187-per-pixel-linked-list-voxelization.md) by relying on hardware rasterizer coverage rather than explicit sorting or linked lists, and from [Triangle-Parallel Solid Voxelization](9770-triangle-parallel-solid-voxelization.md) by using a fragment-shader-driven approach rather than triangle-centric parallel processing.

## How it works
**Inputs and Outputs:**
*   **Input:** A mesh (set of triangles) to be voxelized.
*   **Output:** A 3D voxel grid (density or color values) stored in a texture or buffer.

**Data Structures:**
*   **MSAA Render Target:** A render target texture with an MSAA sample count (typically 8x). This increases video memory usage slightly compared to a non-MSAA target [S1, unofficial].
*   **Voxel Grid:** A 3D texture or structured buffer representing the spatial domain.

**Steps:**
1.  **Setup Render State:** Configure the graphics pipeline to use an MSAA sample count of $N$ (typically $N=8$) [S1, unofficial].
2.  **Bind Render Target:** Bind the MSAA render target to the Output Merger stage [S1, unofficial].
    *   *Note:* In DirectX 11.1+, one may use "Forced Sample Count" to enable MSAA rasterization without binding an MSAA render target, reducing memory overhead [S1, unofficial].
3.  **Rasterization:** Issue draw calls for the mesh. The hardware rasterizer evaluates coverage against the $N$ subsamples per pixel [S1, unofficial].
4.  **Pixel Shader Invocation:** The pixel shader is invoked **once** per pixel if **any** of the $N$ subsamples are covered by the primitive [S1, unofficial].
5.  **Voxel Writing:** Inside the pixel shader, write the voxel data (e.g., density, color) to the voxel grid at the position corresponding to the pixel/fragment. The coverage by any subsample is treated as sufficient evidence that the primitive occupies that voxel space [S1, unofficial].

**Parameters:**
*   **MSAA Count:** Typically 8x subsamples [S1, unofficial]. This provides sufficient spacing to approximate conservative rasterization.

**Pitfalls and Edge Cases:**
*   **Holes:** Primitives lying strictly between subsamples may not be covered by any subsample, leading to "holes" in the voxel grid [S1, unofficial].
*   **Mitigation:** Ensure the voxel grid resolution is sufficiently high relative to the size of the voxelized primitives to minimize the probability of primitives falling between subsamples [S1, unofficial].
*   **Memory:** Binding an MSAA render target increases video memory usage [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Setup phase
msaa_sample_count = 8
render_target = create_msaa_texture(width, height, msaa_sample_count)
pipeline.set_render_target(render_target)
pipeline.set_sample_count(msaa_sample_count)

# Draw phase
for triangle in mesh:
    pipeline.set_geometry(triangle)
    pipeline.draw()

# Pixel Shader (invoked once per pixel if any subsample is covered)
def pixel_shader():
    # Determine voxel coordinates from pixel position
    voxel_pos = get_voxel_coordinates_from_pixel()
    
    # Write to voxel grid. 
    # The fact that this shader was invoked implies coverage by at least one subsample.
    voxel_grid[voxel_pos] = write_voxel_data()
```

## Subtopics

### Conservative Rasterization via Edge Expansion
Expands primitive edges in the Geometry Shader to ensure coverage. More accurate but computationally expensive due to extra geometry processing and potential need for intersection tests to remove redundant voxels [S1, unofficial].

### Hardware Conservative Rasterization
Native support in GPU architectures (e.g., NVIDIA Maxwell and higher). Recommended over MSAA approximation for modern hardware [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(P)$ where $P$ is the number of pixels covered by the mesh. The pixel shader is invoked once per covered pixel, regardless of the number of subsamples covered [S1, unofficial].
*   **Memory Complexity:** $O(W \times H \times N)$ for the MSAA render target, where $W, H$ are screen dimensions and $N$ is the sample count (typically 8) [S1, unofficial].
*   **Strengths:** Simplifies the Geometry Shader by eliminating the need for manual edge expansion; avoids extra pixel shader calls; eliminates the need for post-rasterization primitive-voxel intersection tests to remove redundant voxels [S1, unofficial].
*   **Limitations:** It is an approximation, not genuine conservative rasterization; prone to "holes" if primitives fall between subsamples; requires higher resolution voxel grids to mitigate holes; increases video memory usage due to MSAA target [S1, unofficial].
*   **When to use:** Recommended for hardware prior to NVIDIA Maxwell architecture [S1, unofficial].
*   **When not to use:** On Maxwell or higher architectures, use official conservative rasterization functions instead [S1, unofficial].

## Background
MSAA Voxelization was described as a practical GPU voxelization technique in an unofficial NVIDIA Developer article [S1, unofficial]. It emerged as an alternative to manual conservative rasterization implementations that relied on Geometry Shader edge expansion, which consumed significant GPU cycles [S1, unofficial]. The method leverages existing MSAA hardware features to approximate conservative coverage without additional geometric processing.

## Sources
[S1] Basics of GPU Voxelization, https://developer.nvidia.com/content/basics-gpu-voxelization, fetched 2026-10-28 (unofficial)
