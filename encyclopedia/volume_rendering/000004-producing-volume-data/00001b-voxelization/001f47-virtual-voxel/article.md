# Virtual Voxel


## Overview
Virtual Voxel is a rasterization-based voxel renderer designed for the real-time visualization and modification of large-scale voxel scenes (e.g., $4^{12}$ resolution) by combining virtual texture and virtual geometry concepts [S1]. It streams appropriate levels of detail into GPU memory using a two-level data structure, storing data linearly to enable efficient real-time editing (Union/Subtraction) [S1]. Unlike previous graph-based voxel visualization methods, it employs a rasterization rendering pipeline, resulting in highly efficient modification of large scenes [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A pre-generated Sparse Voxel Octree (STree) representing the scene geometry; camera position and orientation; screen pick position $P$ for modification [S1].
*   **Output:** A 64-bit framebuffer containing position and albedo for opaque voxels; a modified voxel pool reflecting real-time changes [S1].

**Data Structures:**
1.  **STree (Streaming Tree):** A two-level structure [S1].
    *   **High-level:** A 2D uniform grid of chunks [S1].
    *   **Low-level:** A sparse octree within each chunk [S1]. Each node (called a "cell") represents a $64 \times 64 \times 64$ resolution voxel data block [S1].
    *   **Storage:** Only existence flags are recorded for each cell to efficiently store sparse data [S1].
2.  **GPU Voxel Pool:**
    *   Allocates a new page for each streamed cell [S1].
    *   Each page accommodates 16,384 ($128 \times 128$) voxels [S1].
3.  **Visibility Buffer:**
    *   64-bit buffer per pixel [S1].
    *   Layout: `depth_bits << 32 | cell_id << 12 | voxel_id` [S1].
4.  **Framebuffer:**
    *   64-bit buffer per pixel [S1].
    *   Layout: `depth_bits << 32 | albedo` [S1].

**Parameters:**
*   **LOD_factor:** Scalar multiplier for cell size in LOD error calculation. Typical value: 8 (for Berlin Data Set) [S1].

**Algorithm Steps:**

1.  **Streaming (CPU-GPU Handoff):**
    1.  The CPU traverses the STree from the root cell to determine the suitable Level of Detail (LOD) using a distance-based error metric [S1].
    2.  Calculate LOD error:
        $$LOD\_error = \|CellCenterPos - CamPos\| - (LOD\_factor \times CellSize)$$
        where $CellCenterPos$ is the center of the current cell, $CamPos$ is the camera position, and $CellSize$ is the side length of the cell [S1].
    3.  If $LOD\_error > 0$, the CPU evaluates child cells (current cell does not meet LOD metric) [S1].
    4.  If $LOD\_error \le 0$ or the cell is a leaf, send a cell-building request to the GPU [S1].

2.  **Cell Voxelization (GPU):**
    1.  Upon receiving a request, allocate a page in the Voxel Pool for the cell [S1].
    2.  Voxelize the cell at $64^3$ resolution using the method from Crassin and Green [2012] [S1].
    3.  Write the voxelization result into a $64^3$ size buffer [S1].
    4.  Atomically append the result to the allocated page in the Voxel Pool [S1].

3.  **Rendering Pass 1 (Visibility Buffer):**
    1.  Render voxels as points to the 64-bit visibility buffer [S1].
    2.  Use frustum culling to discard invisible cells [S1].
    3.  Each compute thread renders 64 voxels of a cell [S1].

4.  **Rendering Pass 2 (Reconstruction and Final Framebuffer):**
    1.  Reconstruct voxels from the visibility buffer using a method similar to Nanite [S1].
    2.  Each compute thread reads one pixel from the visibility buffer [S1].
    3.  Reconstruct the voxel as a geometry cube with 12 triangles [S1].
    4.  Render the cube; the result contains position and albedo [S1].
    5.  Write the result to the 64-bit framebuffer using an `AtomicMin` operation on the depth component [S1].

5.  **Real-Time Modification (Union/Subtraction):**
    1.  **Screen Pick:** Retrieve the depth value from the framebuffer at the mouse click position $P$ [S1].
    2.  **World Position:** Determine world position $wP$ of $P$ using the inverted camera matrix [S1].
    3.  **AABB Computation:** Compute an Axis-Aligned Bounding Box (AABB) around $wP$ [S1].
    4.  **Cell Identification:** Identify all STree cells intersecting the AABB [S1].
    5.  **Re-voxelization:**
        *   Execute a voxelization shader to re-voxelize the identified cells [S1].
        *   **Subtraction:** Use a buffer containing subtraction cube positions and scales. During voxelization, eliminate voxels residing within these subtraction cubes [S1].
        *   **Union:** Execute a compute shader for the Union operation using union cube inputs. Merge the union result with the regular voxelization result [S1].
    6.  **Update Pool:** Atomically append the combined voxelization result to the allocated voxel page [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

# Constants from [S1]
CELL_RESOLUTION = 64
LOD_FACTOR = 8.0
PAGE_CAPACITY = 16384

def calculate_lod_error(cell_center, camera_pos, cell_size):
    distance = math.sqrt(sum((c - p) ** 2 for c, p in zip(cell_center, camera_pos)))
    return distance - (LOD_FACTOR * cell_size)

def stream_cells(stree_root, camera_pos):
    queue = [stree_root]
    while queue:
        cell = queue.pop(0)
        error = calculate_lod_error(cell.center, camera_pos, cell.size)
        if error > 0 and not cell.is_leaf:
            queue.extend(cell.children)
        else:
            gpu_request_queue.append(cell.id)

def gpu_voxelize_cell(cell_id, geometry_data):
    page_id = voxel_pool.allocate_page()
    # Crassin and Green [2012] rasterization-based voxelization method [G]
    voxel_buffer = crassin_green_voxelize(geometry_data, resolution=CELL_RESOLUTION)
    voxel_pool.append(page_id, voxel_buffer)
    return page_id

def render_visibility_buffer(cells, camera):
    for cell in cells:
        if aabb_frustum_test(cell.aabb, camera.planes):
            voxels = cell.get_voxels()
            for i in range(0, len(voxels), 64):
                batch = voxels[i : i+64]
                for v in batch:
                    screen_pos = perspective_project(v.position, camera)
                    # Layout: depth_bits << 32 | cell_id << 12 | voxel_id [S1]
                    packed = (v.depth << 32) | (cell.id << 12) | v.id
                    visibility_buffer.write(screen_pos, packed)

def render_final_framebuffer(camera):
    for y in range(screen_height):
        for x in range(screen_width):
            packed = visibility_buffer.read(x, y)
            if packed > 0:
                cell_id = (packed >> 12) & 0xFFFFF
                voxel_id = packed & 0xFFF
                voxel_geom = reconstruct_voxel_geometry(cell_id, voxel_id)
                depth, albedo = render_cube_geometry(voxel_geom, camera)
                # Layout: depth_bits << 32 | albedo [S1]
                packed_fb = (depth << 32) | albedo
                framebuffer.atomic_min((x, y), packed_fb)

def modify_scene(mouse_pos, framebuffer, stree, camera):
    depth = framebuffer.get_depth(mouse_pos)
    # World position wP via inverted camera matrix [S1]
    wP = camera.inverted_matrix * mouse_pos
    # Compute AABB around wP [S1]
    aabb = compute_aabb_around(wP)
    affected_cells = stree.find_intersecting_cells(aabb)
    for cell in affected_cells:
        # Subtraction: eliminate voxels inside subtraction cubes [S1]
        # Union: merge union cube voxels [S1]
        new_voxels = voxelize_cell_with_modifications(cell)
        voxel_pool.append(cell.page_id, new_voxels)

# Standard operations [G]
def aabb_frustum_test(aabb, planes):
    return True
def perspective_project(pos, camera):
    return (pos.x, pos.y)
def compute_aabb_around(point):
    return {"min": (point.x - 1.0, point.y - 1.0, point.z - 1.0), "max": (point.x + 1.0, point.y + 1.0, point.z + 1.0)}
def voxelize_cell_with_modifications(cell):
    return cell.voxels
```

**Edge Cases and Pitfalls:**
*   **Transparent Rendering:** Unachievable because the atomic operation (used for depth/visibility) cannot blend color values [S1].
*   **Precision:** The visibility buffer uses a 64-bit layout. Care must be taken to pack `cell_id` and `voxel_id` correctly to avoid overflow or misinterpretation [S1].

## Subtopics

### Streaming LOD
Uses a distance-based error metric to determine which cells to stream [S1].

### Visibility Buffer Rendering
Uses a 64-bit visibility buffer to decouple geometry reconstruction from shading [S1].

### Real-Time Voxel Modification
Supports Union and Subtraction via screen picking and re-voxelization [S1].

## Operating characteristics
*   **Measured Performance:** Average frame time of 6.0459 ms at 4K resolution on a Windows 10 desktop with AMD Threadripper 2950X and RTX4090, measured during a camera fly-through of a 16x16 chunk test scene ($4^{12}$ resolution) [S1].
*   **Strengths:** Real-time rendering of large-scale scenes ($4^{12}$); efficient real-time modification (Union/Subtraction); linear storage allows efficient streaming [S1].
*   **Limitations:** Only supports opaque objects; transparent rendering is unachievable due to atomic operations [S1].

## Background
Virtual Voxel was introduced by Jinyuan Yang and Abraham G. Campbell in 2023 [S1]. It builds on the concepts of Virtual Texture and Virtual Geometry, adapting them for voxel data [S1]. The voxelization method used is based on Crassin and Green [2012] [S1]. The visibility buffer approach references Burns and Hunt [2013] [S1], and the reconstruction method references Nanite (Karis et al. [2021]) [S1]. The method was evaluated using the Berlin Data Set [S1].

## Sources
[S1] VirtualVoxel: Real-Time Large Scale Scene Visualization and Modification, https://doi.org/10.1145/3588028.3603664, fetched 2026-10-30
