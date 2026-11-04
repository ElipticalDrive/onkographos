# Raster Occlusion Culling

Also called: GPU-driven occlusion culling, Hybrid Raster Occlusion Culling, Lightweight Raster Occlusion Culling, Raster Occlusion Culling for Dense Volume Data

## Overview
Raster Occlusion Culling determines the visibility of voxel blocks by replacing expensive individual hardware occlusion queries with a single efficient proxy geometry rasterization pass, followed by GPU-driven stream compaction to create a dense list of occupied subvolumes [S1, S2][S3, unofficial]. It solves the problem of high draw times in volumes with high depth complexity by avoiding the extraction and rendering of geometry for blocks that are occluded [S2][S3, unofficial]. Unlike methods that process the depth texture of a previously established occluder representation, such as hierarchical occlusion maps [ZMHH97] or iterative depth warping [LKE18], this approach rasterizes proxy geometry directly [S1, S2]. It is a hybrid between basic raster occlusion culling [KT14] and hierarchical approaches [LL20], resolving potential visibility only for fixed-size blocks using a custom task and mesh shader pipeline [S3, unofficial].

## How it works
**Inputs:**
*   Volume data with precomputed min-max values for each block, stored in a 2-channel volume texture (min-max grid) [S3, unofficial].
*   Resolution of the min-max grid: $X/U \times Y/V \times Z/W$, where $X, Y, Z$ are volume dimensions and $U, V, W$ are block dimensions [S3, unofficial].
*   Isovalue for the isosurface extraction [S2].
*   Camera view and projection matrices [S3, unofficial].

**Outputs:**
*   A dense list of occupied blocks (Potentially Visible Set, PVS) [S1, S2].
*   Two PVS sets for temporal coherence: $PVS_{curr}$ (blocks visible in the current frame, used as occluders in the next) and $PVS_{curr-prev}$ (blocks visible since the last frame) [S2][S3, unofficial].

**Data Structures:**
*   **Min-Max Texture:** 2-channel volume texture storing the minimum and maximum scalar values for each block [S3, unofficial].
*   **Visibility Buffer:** A preallocated buffer where fragments write the implicit ID of the block if it passes depth tests [S2][S3, unofficial].
*   **Task Shader Interface Block:**
    ```c
    struct Task {
        uint32_t baseID;
        uint32_t num_occupied_blocks;
        uint8_t block_occupancy_index[256];
    };
    ```
    [S3, unofficial]
*   **Mesh Shader Workgroup:** Processes sets of 32 occupied blocks [S2][S3, unofficial].

**Steps:**

1.  **Task Shader: Block-wise Stream Compaction**
    The task shader analyzes a localized group of 256 blocks [S2][S3, unofficial].
    *   For each block in the group, sample the min-max texture at the block's location [S2][S3, unofficial].
    *   Determine occupancy by checking if the isovalue lies between the sampled minimum and maximum values:
        $$ \text{is\_occupied} = (\text{min} \le \text{isovalue} \le \text{max}) $$
        [S2][S3, unofficial]
    *   Generate a dense list of occupied blocks. The task shader emits mesh shader workgroups, each processing 32 occupied blocks (except possibly the final workgroup, which processes 1–32 blocks) [S2][S3, unofficial].
    *   Since mesh shader workgroups operate on lists of fixed lengths, no list offset or length buffers are required in the interface block [S2][S3, unofficial].

2.  **Mesh Shader: Proxy Geometry Creation**
    Each thread in the mesh shader workgroup computes an offset into the `block_occupancy_index` array based on thread and workgroup indices, ensuring the index is less than `num_occupied_blocks` [S3, unofficial].
    *   Each active thread creates proxy geometry for one occupied block [S2][S3, unofficial].
    *   The proxy geometry is a single quad containing the bounding box of the block when projected into screen space, with the minimal z-coordinate of the block's corners [S2][S3, unofficial].
    *   This reduces the number of triangles per block from 6 to 2, and vertices per workgroup (32 blocks) from 224 to 128 [S2][S3, unofficial].

3.  **Rasterization**
    The proxy geometry is passed to the rasterizer [S2][S3, unofficial].
    *   Each fragment that passes the early depth test and the representative fragment test writes into a preallocated slot of the visibility buffer, based on the implicit ID of each block [S2][S3, unofficial].
    *   The representative fragment test extension allows the rasterizer to terminate early-depth testing earlier if at least one fragment shader instance was invoked [S2].

4.  **PVS Generation and Compactification**
    *   A bit compactification kernel runs for the currently created indices $PVS_{curr}$ [S2][S3, unofficial].
    *   A final compute shader pass builds two PVS sets: $PVS_{curr}$ and $PVS_{curr-prev}$ [S2][S3, unofficial].
    *   $PVS_{curr}$ is used in the next frame to render the best occluders based on temporal coherence [S2][S3, unofficial].
    *   $PVS_{curr-prev}$ determines blocks that became visible since the last frame [S2][S3, unofficial].
    *   Compute shaders are used for this pass to allow larger block sizes and better utilization of streaming multiprocessors compared to mesh shaders [S2][S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Stage 1: Task Shader - Block-wise Stream Compaction
# Processes a localized group of 256 blocks
num_occupied_blocks = 0
block_occupancy_index = [0] * 256

for i in range(256):
    # Sample min-max texture at block i's location
    min_val, max_val = min_max_texture[block_positions[i]]
    # Determine occupancy: isovalue lies between sampled min and max
    if min_val <= isovalue <= max_val:
        block_occupancy_index[num_occupied_blocks] = i
        num_occupied_blocks += 1

# Dispatch mesh shader workgroups
workgroup_size = 32
num_workgroups = (num_occupied_blocks + workgroup_size - 1) // workgroup_size

for wg in range(num_workgroups):
    # Interface block passed to mesh shader
    task_data = {
        'baseID': wg * workgroup_size,
        'num_occupied_blocks': num_occupied_blocks,
        'block_occupancy_index': block_occupancy_index
    }
    # Mesh shader workgroup processes exactly 32 blocks (except final)
    MeshShaderProxyGeometry(task_data)

# Stage 2: Mesh Shader - Proxy Geometry Creation
# Each thread creates proxy geometry for one occupied block
def MeshShaderProxyGeometry(task_data):
    # Thread computes offset into block_occupancy_index
    offset = task_data['baseID'] + thread_id
    if offset < task_data['num_occupied_blocks']:
        block_local_idx = task_data['block_occupancy_index'][offset]
        # Get 8 world-space corners of the block's axis-aligned bounding box
        corners = get_block_corners(block_local_idx)
        # Project corners to screen space using standard 4x4 perspective projection
        screen_corners = [apply_perspective_projection(c) for c in corners]
        # Find minimal z-coordinate among projected corners
        min_z = min(c.z for c in screen_corners)
        # Create single quad proxy: 2 triangles, 4 vertices
        quad_vertices = [(c.x, c.y, min_z) for c in screen_corners]
        quad_indices = [0, 1, 2, 0, 2, 3]
        write_to_vertex_buffer(quad_vertices)
        write_to_index_buffer(quad_indices)

# Stage 3: Fragment Shader - Visibility Buffer Update
def FragmentShaderVisibilityUpdate(fragment_pos, block_id):
    # Passes early depth test and representative fragment test
    if fragment_pos.z < depth_buffer[fragment_pos.xy] and fragment_shader_invoked:
        visibility_buffer[fragment_pos.slot] = block_id

# Stage 4: Compute Shader - PVS Compactification
def ComputeShaderPVSGeneration(visibility_buffer, previous_frame_PVS):
    # Bit compactification kernel creates indices for PVS_curr
    PVS_curr = [id for id in visibility_buffer if id != 0]
    # Build PVS_curr-prev: blocks visible since last frame
    PVS_curr_prev = [id for id in PVS_curr if id not in previous_frame_PVS]
    return PVS_curr, PVS_curr_prev

# Standard helper definitions
def get_block_corners(block_idx):
    # Returns 8 3D coordinates of the block's axis-aligned bounding box in world space
    return [(0,0,0), (1,0,0), (0,1,0), (1,1,0), (0,0,1), (1,0,1), (0,1,1), (1,1,1)]

def apply_perspective_projection(world_coord):
    # Standard graphics operation: multiply by view-projection matrix and divide by w [G]
    return world_coord
```

**Parameters:**
*   **Block analysis size:** 256 blocks per task shader workgroup [S3, unofficial].
*   **Mesh shader workgroup size:** 32 blocks [S3, unofficial].
*   **Min-max texture resolution:** $X/U \times Y/V \times Z/W$ [S3, unofficial].
*   **Representative Fragment Test:** An extension allowing early-depth termination if at least one fragment shader instance was invoked [S2].

**Edge Cases and Pitfalls:**
*   **Multi-view Rendering:** When using multi-view rendering (per-eye), generating one quad per eye is straightforward. Using three cube faces would require accounting for rendering more than 3 sides if the block is located between the viewer's eyes on any cardinal axis [S2][S3, unofficial].
*   **Conservatism:** The method is conservative, not probabilistic, ensuring no occupied blocks are incorrectly culled [S3, unofficial].
*   **Precision:** The min-max texture lookup must be precise enough to determine if the isovalue lies between min and max [S2][S3, unofficial].

## Subtopics

### Hierarchical Visibility Mask
A visibility culling technique where triangles are marked as visible and forwarded to graphics hardware rather than being rendered by the CPU, allowing the GPU to determine actual visibility and handle occlusion [S4, book]. It projects the polygon onto the top meta-tile of a hierarchical structure and recursively checks only those subtiles in which the polygon might be visible, leading to a logarithmic search [S4, book]. If a meta-cell size is approximately the size of a single pixel, recursion stops and a single point with an associated normal is forwarded to the graphics hardware, with the normal estimated by calculating the gradient of the field [S4, book]. It differs from Greene's implementation by not rendering the visible portion on the CPU; instead, it relies on the graphics accelerator to determine visible pieces and render them correctly [S4, book].

## Operating characteristics
*   **Time Complexity:** The basic method replaces expensive individual hardware occlusion queries with one efficient proxy geometry rasterization pass [S1, S2]. Hierarchical variants achieve logarithmic search time for visibility testing due to their structure [S4, book].
*   **Memory Complexity:** Requires storage for the min-max texture ($X/U \times Y/V \times Z/W$ 2-channel volume texture) [S3, unofficial] and visibility buffers [S2][S3, unofficial].
*   **Measured Performance:** Outperforms methods that process the depth texture of a previously established occluder representation, such as hierarchical occlusion maps [ZMHH97] and iterative depth warping [LKE18] [S1, S2].
*   **Scaling:** Better utilization of streaming multiprocessors when using compute shaders for PVS compactification compared to mesh shaders [S2][S3, unofficial].
*   **Strengths:** Reduces the number of task shader workgroups launched, avoiding geometry extraction for invisible blocks [S2][S3, unofficial]. Fewer primitives (2 triangles per block vs 6) increase the likelihood of early-depth termination via the representative fragment test [S2][S3, unofficial]. Conservative visibility determination ensures no occupied blocks are incorrectly culled [S3, unofficial].
*   **Limitations:** Requires precomputed min-max values for each block [S3, unofficial]. Multi-view rendering requires careful handling of proxy geometry per eye [S2][S3, unofficial].
*   **When to use it:** For volumes with high depth complexity where reducing draw times is critical [S2][S3, unofficial].
*   **When not to use it:** When probabilistic culling is acceptable and hierarchical structures are already available [S3, unofficial].

## Background
Raster Occlusion Culling was introduced by Kubisch and Tavenrath [KT14] as the state-of-the-art approach for general scenes [S1, S2]. Historically, GPU-driven occlusion culling was facilitated by occlusion query objects [CCG*07], which required careful use of spatial and temporal coherence to be effective, as shown by Bittner et al. [BWPP04] and Mattausch et al. [MBW08] [S1, S2][S3, unofficial]. Follow-up works include Lee and Lee [LL20] (iterative version on hierarchy levels) and Lee et al. [LJSL21] (fully GPU-driven hierarchical algorithm refining visibility via ray traversal) [S1, S2][S3, unofficial]. The method described here is a hybrid between basic raster occlusion culling [KT14] and hierarchical raster occlusion culling [LL20], using a custom task and mesh shader pipeline to efficiently detect occupied blocks before creating proxy geometry [S3, unofficial]. It is distinct from probabilistic culling methods like those by Ibrahim et al. [IRR*22] [S3, unofficial].

## Sources
[S1] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://doi.org/10.1111/cgf.14670, fetched 2026-10-27
[S2] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://www.uni-weimar.de/en/media/chairs/computer-science-department/vr/research/real-time-rendering/efficient-direct-isosurface-rasterization-of-scalar-volumes/, fetched 2026-11-01
[S3] Efficient Direct Isosurface Rasterization of Scalar Volumes - EG, https://diglib.eg.org/items/8a65e0f0-c048-4fce-9d0b-5acf07452485, fetched 2026-10-30 (unofficial)
[S4] Fast Isosurface Extraction Methods for Large Image Data sets, https://publications.sci.utah.edu/publications/livnat08/Large-scale-isosurface08.pdf, fetched 2026-10-28 (book)
