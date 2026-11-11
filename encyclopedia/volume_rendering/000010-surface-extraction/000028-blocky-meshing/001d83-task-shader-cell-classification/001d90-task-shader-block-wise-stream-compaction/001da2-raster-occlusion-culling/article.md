# Raster Occlusion Culling

Also called: Depth Buffer Occlusion Culling, GPU-driven occlusion culling, Hybrid Raster Occlusion Culling, Lightweight Raster Occlusion Culling, Raster Occlusion Culling for Dense Volume Data

## Overview
Raster Occlusion Culling is a GPU-driven visibility determination technique that identifies a potentially visible set (PVS) of voxel blocks by rasterizing lightweight proxy geometry, replacing expensive individual hardware occlusion queries with a single efficient rasterization pass followed by GPU-driven stream compaction [S1, S3][S4, unofficial]. It solves the problem of high draw times in scenes with high depth complexity by avoiding the extraction and rendering of geometry for blocks that are occluded by closer volume data [S3][S4, unofficial]. Unlike its siblings, it operates on a flat list of fixed-size blocks rather than a hierarchical structure, using a custom task and mesh shader pipeline to detect occupancy before generating proxy geometry [S4, unofficial]. It outperforms methods that process depth textures of previously established occluder representations, such as hierarchical occlusion maps [ZMHH97] and iterative depth warping [LKE18] [S1, S3].

## How it works
**Inputs and Outputs:**
- **Input:** A min-max texture storing the minimum and maximum scalar values for each voxel block; the current camera view matrix and projection matrix; the isovalue for isosurface extraction [S3][S4, unofficial].
- **Output:** A dense list of indices representing the PVS ($PVS_{curr}$) of occupied blocks, and optionally a secondary list ($PVS_{curr-prev}$) of blocks visible since the previous frame [S3][S4, unofficial].

**Data Structures:**
1.  **Min-Max Texture:** A 2-channel volume texture with resolution $\frac{X}{U} \times \frac{Y}{V} \times \frac{Z}{W}$, where $X, Y, Z$ are the volume dimensions and $U, V, W$ are the block dimensions [S4, unofficial]. Each texel stores the min and max values of the corresponding block [S4, unofficial].
2.  **Visibility Buffer:** A preallocated buffer (e.g., image buffer or structured buffer) where fragments write the implicit ID of the block if it passes depth tests [S3][S4, unofficial].
3.  **Task Shader Interface Block:**
    ```glsl
    struct Task {
        uint32_t baseID;
        uint32_t num_occupied_blocks;
        uint8_t block_occupancy_index[256];
    };
    ```
    This structure is passed from the task shader to the mesh shader [S4, unofficial]. The `block_occupancy_index` array contains flags for 256 blocks, indicating which are occupied [S4, unofficial].

**Numbered Steps:**

1.  **Task Shader: Block-wise Stream Compaction** [S3]
    The task shader analyzes a localized group of 256 blocks [S3][S4, unofficial].
    1.  For each block in the group, sample the min-max texture at the block's location [S3][S4, unofficial].
    2.  Determine occupancy: if the isovalue lies between the sampled minimum and maximum values, the block is marked as occupied [S3][S4, unofficial].
    3.  Generate a dense list of occupied blocks [S3]. The task shader creates mesh shader workgroups that process sets of 32 occupied blocks [S3][S4, unofficial].
    4.  Populate the `block_occupancy_index` array with the indices of occupied blocks [S4, unofficial].
    5.  Dispatch mesh shader workgroups. Each workgroup operates on a fixed length of 32 blocks (except possibly the final workgroup, which handles 1–32 blocks) [S3][S4, unofficial]. No list offset or length buffers are required in the interface block because of the fixed-length operation [S3][S4, unofficial].

2.  **Mesh Shader: Proxy Geometry Creation**
    Each active thread in the mesh shader workgroup computes an offset into the `block_occupancy_index` array based on thread and workgroup indices, ensuring the index is less than `num_occupied_blocks` [S4, unofficial].
    1.  For each occupied block, create proxy geometry: a single quad representing the block's bounding box projected into screen space, using the minimal z-coordinate of the block's corners [S3][S4, unofficial].
    2.  This reduces the geometry from 6 triangles (3 visible cube faces, 224 vertices for 32 blocks) to 2 triangles (1 quad, 128 vertices for 32 blocks) [S3][S4, unofficial].
    3.  Write the vertex and index buffers for the quad proxies [S3][S4, unofficial].

3.  **Rasterization and Visibility Buffer Update** [S4, unofficial]
    1.  Rasterize the proxy geometry [S3][S4, unofficial].
    2.  Perform standard rendering with `glDepthFunc(GL_LEQUAL)` to discard fragments hidden by closer fragments [S2, unofficial].
    3.  Leverage the **Representative Fragment Test** extension [KTB*18], which allows the rasterizer to terminate early-depth testing earlier if at least one fragment shader instance was invoked [S3][S4, unofficial]. Rasterizing fewer primitives (quads vs. cube faces) increases the likelihood of this early termination [S3][S4, unofficial].
    4.  For each fragment that passes the early depth test and representative fragment test, write the implicit ID of the block into a preallocated slot of the visibility buffer [S3][S4, unofficial].

4.  **PVS Generation (Compute Shader)**
    1.  Run a bit compactification kernel on the visibility buffer to create a dense list of indices for $PVS_{curr}$ [S3][S4, unofficial].
    2.  Execute a final compute shader pass to build two PVS sets:
        -   $PVS_{curr}$: Used in the next frame to render the best occluders based on temporal coherence [S3][S4, unofficial].
        -   $PVS_{curr-prev}$: Determines blocks that became visible since the last frame [S3][S4, unofficial].
    3.  Use a compute shader instead of mesh shaders for this stage to allow larger block sizes and better utilization of streaming multiprocessors [S4, unofficial].

**Parameters:**
-   **Block Analysis Size:** 256 blocks per task shader workgroup [S4, unofficial].
-   **Mesh Shader Workgroup Size:** 32 blocks per workgroup [S4, unofficial].
-   **Min-Max Texture Resolution:** $\frac{X}{U} \times \frac{Y}{V} \times \frac{Z}{W}$ [S4, unofficial].

**Edge Cases and Pitfalls:**
-   **Multi-view Rendering:** When using multi-view rendering (per-eye), generating one quad per eye is straightforward [S3]. If using three cube faces, one must account for rendering more than 3 sides if the block is located between the viewer's eyes on any cardinal axis [S3][S4, unofficial].
-   **Conservatism:** The method is conservative, not probabilistic, ensuring no occupied blocks are incorrectly culled [S4, unofficial].
-   **Precision:** The min-max texture lookup must be precise enough to distinguish blocks where the isovalue is near the min or max values [S3][S4, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

# Data Structures
class TaskStruct:
    def __init__(self):
        self.baseID = 0
        self.num_occupied_blocks = 0
        self.block_occupancy_index = [0] * 256

def get_block_coord(linear_idx, block_dims):
    # Converts linear index to 3D block coordinates
    z = linear_idx // (block_dims[0] * block_dims[1])
    y = (linear_idx % (block_dims[0] * block_dims[1])) // block_dims[0]
    x = (linear_idx % (block_dims[0] * block_dims[1])) % block_dims[0]
    return (x, y, z)

def sample_min_max_texture(coord, min_max_texture):
    # Standard 3D texture lookup returning (min_val, max_val)
    return min_max_texture[coord]

def create_quad_proxy(block_coord, block_dims, view_proj_matrix):
    # Sources do not provide projection math; standard view-projection multiplication applies [G]
    return []

def run_raster_occlusion_culling(volume_dims, block_dims, isovalue, min_max_texture, view_proj_matrix):
    # Calculate total blocks
    num_blocks_x = volume_dims[0] // block_dims[0]
    num_blocks_y = volume_dims[1] // block_dims[1]
    num_blocks_z = volume_dims[2] // block_dims[2]
    total_blocks = num_blocks_x * num_blocks_y * num_blocks_z
    visibility_buffer = [0] * total_blocks

    # 1. Task Shader: Block-wise Stream Compaction
    task_struct = TaskStruct()
    for i in range(256):
        coord = get_block_coord(i, block_dims)
        min_val, max_val = sample_min_max_texture(coord, min_max_texture)
        if isovalue >= min_val and isovalue <= max_val:
            task_struct.block_occupancy_index[i] = 1
            task_struct.num_occupied_blocks += 1

    # 2. Mesh Shader: Proxy Geometry Creation
    quad_list = []
    for i in range(256):
        if task_struct.block_occupancy_index[i] == 1:
            coord = get_block_coord(i, block_dims)
            quad_list.append(create_quad_proxy(coord, block_dims, view_proj_matrix))

    # 3. Rasterization & Visibility Buffer Update
    # Rasterize proxy geometry with glDepthFunc(GL_LEQUAL)
    # Representative Fragment Test extension allows early-depth termination
    # if at least one fragment shader instance was invoked.
    for block_id, quad in enumerate(quad_list):
        # Simulate fragment shader invocation and depth test
        # If fragment passes early depth test and representative fragment test:
        visibility_buffer[block_id] = 1

    # 4. Compute Shader: PVS Generation
    # Bit compactification kernel creates dense list of indices
    PVS_curr = [i for i, val in enumerate(visibility_buffer) if val == 1]
    # Final compute shader pass builds PVS_curr-prev (blocks visible since last frame)
    PVS_curr_prev = [] # Implementation depends on temporal coherence comparison
    return PVS_curr, PVS_curr_prev
```

## Subtopics

### [Occlusion Culling Granularity](1671-occlusion-culling-granularity.md)
Configures the spatial resolution for raster occlusion culling to match the leaf-level granularity of a reference ray-marching octree. It ensures that the culling proxy geometry aligns with the underlying extraction data structures for direct performance comparison. Choose this variant when precise granularity matching between culling and extraction is required.

### [Volumetric Occlusion Culling via Occlusion Queries](4594-volumetric-occlusion-culling-via-occlusion-queries.md)
Determines if a voxel block is visible by rasterizing its bounding box and checking for fragment depth test passes. It differs from standard binary occlusion culling by incorporating accumulated opacity into the depth test, ensuring that blocks behind fully opaque foreground volumes are correctly culled. Choose this variant when opacity-aware culling is necessary.

### [Hierarchical Raster Occlusion Culling](6061-hierarchical-raster-occlusion-culling.md)
A fully GPU-driven technique that reduces rasterization overhead by performing visibility tests on a coarse hierarchy of group nodes rather than individual leaf objects, refining results via ray traversal to the leaf level. It distinguishes itself from standard Raster Occlusion Culling by replacing the exhaustive rasterization of all leaf proxies with a hierarchical culling pass, and from Iterative Raster Occlusion Culling by utilizing a hybrid raster-ray approach for full hierarchy descent. Choose this variant for scenes with deep hierarchies to minimize rasterization costs.

### Hierarchical Visibility Mask
A visibility culling technique where triangles are marked as visible and forwarded to graphics hardware rather than being rendered by the CPU, allowing the GPU to determine actual visibility and handle occlusion [S5, book]. It projects the polygon onto the top meta-tile of a hierarchical structure and recursively checks only those subtiles in which the polygon might be visible, leading to a logarithmic search [S5, book]. If a meta-cell size is approximately the size of a single pixel, it stops recursion and forwards a single point with an associated normal to the graphics hardware [S5, book]. The normal is estimated by calculating the gradient of the field [S5, book]. It differs from Greene's implementation by not rendering the visible portion on the CPU; instead, it relies on the graphics accelerator to determine visible pieces and render them correctly [S5, book]. Choose this variant when CPU rendering overhead must be avoided [S5, book].

### Cluster Visibility Buffer Culling
A culling technique using a visibility buffer to prune non-visible clusters, performing back-face and occlusion culling without requiring a hierarchical Z-buffer or storing cluster bounds/normal cones [S6]. It avoids full cell rasterisation of clusters that are fully occluded or backfacing by storing visibility bits per cluster [S6]. Choose this variant when working with clustered grid cell data structures and deferred splatting pipelines [S6].

## Operating characteristics
-   **Time Complexity:** The culling pass is dominated by the rasterization of proxy geometry and the subsequent stream compaction [S4, unofficial]. The task shader processes blocks in groups of 256, and the mesh shader processes 32 blocks per workgroup [S4, unofficial]. The compute shader performs bit compactification, which is efficient for dense lists [S4, unofficial].
-   **Memory Complexity:** Requires storage for the min-max texture ($\frac{X}{U} \times \frac{Y}{V} \times \frac{Z}{W}$ 2-channel volume texture) [S4, unofficial], the visibility buffer, and the PVS lists [S3][S4, unofficial].
-   **Measured Performance:** Outperforms methods that process depth textures of previously established occluder representations, such as hierarchical occlusion maps [ZMHH97] and iterative depth warping [LKE18] [S1, S3]. It is effective for scenes with high depth complexity where the fragment shader is heavy, as it reduces the number of task shader workgroups launched [S3][S4, unofficial].
-   **Scaling:** Not scalable for very high polygon counts in general scenes, but effective for volume data because the fragment shader cost dominates [S2, unofficial]. The method is conservative, ensuring no occupied blocks are incorrectly culled, unlike probabilistic methods [S4, unofficial].
-   **Strengths:** Low draw times for high depth complexity; efficient use of GPU hardware via task/mesh shaders; leverages early-depth termination via representative fragment test [S3][S4, unofficial].
-   **Limitations:** Requires precomputed min-max textures [S4, unofficial]; conservative culling may still process some occluded blocks if the min-max bounds are loose [S3][S4, unofficial].
-   **When to use:** When rendering dense volume data with high depth complexity and heavy fragment shaders [S2, unofficial][S3].
-   **When not to use:** When probabilistic culling is acceptable and hierarchical structures are already available for other purposes [S4, unofficial].

## Background
Raster Occlusion Culling was introduced by Kubisch and Tavenrath [KT14] as the state-of-the-art approach for efficient visibility determination for general scenes [S1, S3]. It replaced earlier GPU-driven occlusion culling methods that relied on occlusion query objects [CCG*07], which required careful exploitation of spatial and temporal coherence [S1, S3]. Follow-up works include Lee and Lee's [LL20] iterative version on hierarchy levels and Lee et al.'s [LJSL21] fully GPU-driven hierarchical algorithm refining visibility via ray traversal [S1, S3]. The method described here is a hybrid between basic raster occlusion culling [KT14] and hierarchical raster occlusion culling [LL20], resolving potential visibility only for fixed-size blocks using a custom task and mesh shader pipeline [S4, unofficial].

## Sources
[S1] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://doi.org/10.1111/cgf.14670, fetched 2026-10-27
[S2] Voxel Cone Tracing - GitHub, https://raw.githubusercontent.com/righier/gidemo/master/report.pdf, fetched 2026-10-27 (unofficial)
[S3] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://www.uni-weimar.de/en/media/chairs/computer-science-department/vr/research/real-time-rendering/efficient-direct-isosurface-rasterization-of-scalar-volumes/, fetched 2026-11-01
[S4] Efficient Direct Isosurface Rasterization of Scalar Volumes - EG, https://diglib.eg.org/items/8a65e0f0-c048-4fce-9d0b-5acf07452485, fetched 2026-10-30 (unofficial)
[S5] Fast Isosurface Extraction Methods for Large Image Data sets, https://publications.sci.utah.edu/publications/livnat08/Large-scale-isosurface08.pdf, fetched 2026-10-28 (book)
[S6] Clustered Grid Cell Data Structure for Isosurface Rendering, https://doi.org/10.24132/jwscg.2020.28.2, fetched 2026-10-26
