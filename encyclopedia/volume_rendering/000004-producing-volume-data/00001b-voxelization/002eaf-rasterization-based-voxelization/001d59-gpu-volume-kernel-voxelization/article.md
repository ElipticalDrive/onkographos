# GPU Volume Kernel Voxelization

Also called: Compute Shader Voxelization, Parallel XY Voxelization

## Overview
GPU Volume Kernel Voxelization is a technique that accelerates the voxelization of triangle meshes by parallelizing the iteration over the 2D XY plane of the voxel grid using a GPU compute shader [S1]. It solves the performance bottleneck of CPU-based voxelization, where nested loops over X and Y coordinates are executed sequentially [S1]. Unlike [Rasterization-based Voxelization](908-rasterization-based-voxelization.md) and [3D Grid Rasterization](903-3d-grid-rasterization.md) which project geometry into 3D textures, this method retains the explicit triangle-voxel intersection logic of CPU implementations but distributes the workload across GPU threads, achieving significant speedups (e.g., 50x) for high-resolution grids [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangle mesh (vertices and indices), voxel grid dimensions ($Width, Height, Depth$), and voxel unit size [S1].
*   **Output:** A `GPUVoxelData` object containing the populated voxel buffer and grid metadata [S1].

**Data Structures:**
*   **Voxel Buffer:** A 1D array or structured buffer representing the 3D grid [S1]. Access is performed via a helper function `get_voxel_index(x, y, z)` which maps 3D coordinates to the buffer offset [S1].
*   **Mesh Buffers:** Vertex and triangle buffers (`vertBuffer`, `triBuffer`) are bound to the compute shader context before dispatch and released afterward [S1].

**Parameters:**
*   **Thread Group Size:** 8×8×1 threads per group ($ThreadX=8, ThreadY=8, ThreadZ=1$) [S1].
*   **Grid Limits:** `_Width`, `_Height`, and `_Depth` define the maximum bounds of the voxel grid [S1].

**Algorithm Steps:**
1.  **Initialization:** Create a compute shader kernel named "Volume" and bind the voxel buffer to it using a set buffer command [S1].
2.  **Dispatch Calculation:** Calculate the number of thread groups to dispatch [S1]. The X and Y group counts are derived from dividing the grid width and height by the thread group dimensions, adding 1 to each to handle remainders (ceiling division) [S1].
    *   $GroupsX = \lfloor Width / ThreadX \rfloor + 1$ [S1].
    *   $GroupsY = \lfloor Height / ThreadY \rfloor + 1$ [S1].
    *   $GroupsZ = ThreadZ$ [S1].
3.  **Kernel Execution:** Dispatch the compute shader with the calculated group counts [S1].
4.  **Per-Thread Processing (Inside Kernel):**
    *   Retrieve the global thread ID $(x, y, z)$ from `SV_DispatchThreadID` [S1].
    *   Clamp or check bounds: If $x \ge Width$ or $y \ge Height$, return immediately [S1].
    *   Iterate through all Z depths from 0 to $Depth - 1$ [S1].
    *   For each Z, calculate the voxel index and retrieve the voxel data from the buffer [S1].
    *   Perform the voxelization logic (triangle-voxel intersection) similar to the CPU implementation [S1].
5.  **Cleanup:** Release the mesh data buffers (`vertBuffer`, `triBuffer`) as they are no longer needed after the kernel completes [S1].
6.  **Return:** Construct and return a `GPUVoxelData` object containing the voxel buffer, dimensions, and unit size [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Compute Shader Kernel: Volume
# Dispatched with groups calculated as:
# groups_x = (width // 8) + 1
# groups_y = (height // 8) + 1
# groups_z = 1

def Volume(id_x, id_y, id_z, Width, Height, Depth, VoxelBuffer):
    x = int(id_x)
    y = int(id_y)
    if x >= Width:
        return
    if y >= Height:
        return
    for z in range(Depth):
        # [G] get_voxel_index(x, y, z) maps 3D coordinates to buffer offset; implementation not provided in sources.
        voxel_index = get_voxel_index(x, y, z)
        voxel = VoxelBuffer[voxel_index]
        # [G] The source states processing continues "Almost the same as in CPUVoxelizer.Voxelize",
        # but does not provide the specific triangle-voxel intersection algorithm details.

def run_gpu_voxelization(mesh_vertices, mesh_indices, Width, Height, Depth, Unit, VoxelBuffer):
    # 1. Initialize kernel and bind buffers
    # [G] Create compute shader kernel "Volume" and bind VoxelBuffer to "_VoxelBuffer"
    # 2. Dispatch
    ThreadX = 8
    ThreadY = 8
    ThreadZ = 1
    GroupsX = (Width // ThreadX) + 1
    GroupsY = (Height // ThreadY) + 1
    GroupsZ = 1
    # [G] Dispatch compute shader with groups (GroupsX, GroupsY, GroupsZ)
    # 3. Cleanup
    # [G] Release mesh vertex and index buffers (vertBuffer, triBuffer)
    # 4. Return result
    return GPUVoxelData(VoxelBuffer, Width, Height, Depth, Unit)
```

**Edge Cases and Pitfalls:**
*   **Remainder Handling:** The dispatch calculation adds 1 to the X and Y group counts to ensure coverage of grid dimensions that are not exact multiples of the thread group size [S1].
*   **Bounds Checking:** Explicit checks (`if x >= _Width`) are required inside the kernel because the dispatch may launch threads beyond the valid grid range due to the ceiling division [S1].
*   **Memory Management:** Mesh buffers must be explicitly released after the kernel execution to free GPU memory, as they are not automatically garbage collected in some contexts [S1].

## Operating characteristics
*   **Performance:** Under test conditions (Windows 10, Core i7, 32GB RAM, GeForce GTX 980, mesh with 5319 vertices/9761 triangles, resolution 256), the GPU implementation runs approximately 50 times faster than the CPU implementation [S1].
*   **Scaling:** Performance depends heavily on the execution environment, the number of polygons in the mesh, and the voxel resolution [S1].
*   **Strengths:** Significant speedup for real-time applications; leverages GPU parallelism for the XY iteration loop [S1].
*   **Limitations:** Coloring/texture application is more difficult compared to rendering-based methods; the method relies on explicit triangle-voxel intersection which can be computationally intensive per voxel [S1].
*   **When to use:** When real-time voxelization of complex meshes is required and the primary bottleneck is the iteration over the voxel grid rather than the intersection calculation itself [S1].
*   **When not to use:** When accurate texture mapping or coloring of voxels is a primary requirement, as rendering-based methods (projecting to 3D texture) are easier for coloring [S1].

## Background
This method was introduced in the context of real-time GPU-based voxelization for Unity, specifically to accelerate the voxelization of mesh models for applications like particle systems [S1]. It builds upon the concept of CPU-based voxelization by replacing the sequential XY loops with a parallel compute shader kernel [S1]. The approach contrasts with methods that construct voxel data by rendering the model from XYZ directions into a 3D texture via parallel projection [S1].

## Sources
[S1] Real-Time GPU-Based Voxelizer - GitHub Pages, https://freder.github.io/UnityGraphicsProgrammingBook1/html-translated/vol2/Chapter%201%20_%20Real-Time%20GPU-Based%20Voxelizer.html, fetched 2026-10-27
