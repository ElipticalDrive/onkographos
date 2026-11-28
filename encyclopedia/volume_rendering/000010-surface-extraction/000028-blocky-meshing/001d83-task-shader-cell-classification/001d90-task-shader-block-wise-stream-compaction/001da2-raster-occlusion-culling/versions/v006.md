# Raster Occlusion Culling

Also called: Depth Buffer Occlusion Culling, GPU-driven occlusion culling, Hybrid Raster Occlusion Culling, Lightweight Raster Occlusion Culling, Raster Occlusion Culling for Dense Volume Data

## Overview
Raster Occlusion Culling is a GPU-driven technique that determines the visibility of voxel blocks by rasterizing lightweight proxy geometry (typically quads) and using the resulting depth buffer to identify a Potentially Visible Set (PVS) [S1, S3][S4, unofficial]. It replaces expensive individual hardware occlusion queries with a single efficient proxy rasterization pass followed by GPU-driven stream compaction [S1, S3]. This method sets the PVS for subsequent isosurface extraction stages, avoiding the generation of geometry for blocks that are fully occluded by closer volumes [S3][S4, unofficial]. It differs from its sibling [Task Shader Block-wise Stream Compaction](1540-task-shader-block-wise-stream-compaction.md) by performing the actual depth-based visibility test via rasterization rather than purely via compute-based occupancy checks, and from [GPU-driven Visibility Culling](3640-gpu-driven-visibility-culling.md) by relying on standard rasterization hardware for occlusion testing rather than ray tracing or other visibility acceleration structures.

## How it works
The method operates in a pipeline involving task shaders, mesh shaders, rasterization, and compute shaders to filter voxel blocks.

### Inputs and Outputs
*   **Input:** A volume dataset with a precomputed min-max texture (2-channel volume texture storing min/max values per block) [S4, unofficial]. The resolution of this texture is $X/U \times Y/V \times Z/W$, where $X, Y, Z$ are volume dimensions and $U, V, W$ are block dimensions [S4, unofficial].
*   **Output:** A dense list of indices representing the PVS of occupied blocks ($PVS_{curr}$) and optionally a list of blocks visible since the last frame ($PVS_{curr-prev}$) [S3][S4, unofficial].

### Data Structures
1.  **Min-Max Texture:** A 2-channel volume texture (e.g., `RG32UI` or similar format depending on data type) storing the minimum and maximum scalar values for each voxel block [S4, unofficial].
2.  **Visibility Buffer:** A buffer (often an image or structured buffer) where fragments write the implicit ID of the block they belong to if they pass depth tests [S3][S4, unofficial].
3.  **Task Shader Interface:**
    ```python
    struct Task {
        uint32_t baseID;
        uint32_t num_occupied_blocks;
        uint8_t block_occupancy_index[256];
    };
    ```
    This structure is passed to the mesh shader. It does not require list offset or length buffers because mesh shader workgroups operate on lists of fixed lengths (32 blocks) [S3][S4, unofficial].
4.  **PVS Lists:** Arrays of indices storing the compacted list of visible block IDs [S3][S4, unofficial].

### Steps

1.  **Task Shader: Block-wise Stream Compaction**
    *   The task shader analyzes a localized group of 256 blocks [S3][S4, unofficial].
    *   For each block, sample the min-max texture at the block's location [S3][S4, unofficial].
    *   Determine occupancy by checking if the target isovalue lies between the sampled minimum and maximum values:
        $$ min\_val \leq isovalue \leq max\_val $$
    *   If occupied, add the block's index to a local occupancy list [S3][S4, unofficial].
    *   Generate a dense list of occupied blocks. Emit mesh shader workgroups, each processing a fixed set of 32 occupied blocks (the final workgroup may process 1–32 blocks) [S3][S4, unofficial].

2.  **Mesh Shader: Proxy Geometry Creation**
    *   Each active thread in the mesh shader computes an offset into the `block_occupancy_index` array based on thread and workgroup indices, ensuring the index is less than `num_occupied_blocks` [S4, unofficial].
    *   Each active thread creates proxy geometry for one occupied block [S3][S4, unofficial].
    *   **Proxy Geometry:** A single quad representing the bounding box of the block projected into screen space, using the minimal z-coordinate of the block's corners [S3][S4, unofficial].
        *   This reduces triangles from 6 to 2 per block (compared to Kubisch's 3 visible cube faces) [S3][S4, unofficial].
        *   Vertices per workgroup: 128 (vs 224 for 32 blocks in Kubisch) [S3][S4, unofficial].
        *   Primitives per workgroup: 64 [S3][S4, unofficial].
    *   Write vertex and index buffers for the quad proxies [S3][S4, unofficial].

3.  **Rasterization**
    *   Rasterize the proxy geometry [S3][S4, unofficial].
    *   Perform standard depth testing with `glDepthFunc(GL_LEQUAL)` (or equivalent) [S2, unofficial].
    *   Utilize the **Representative Fragment Test** extension [KTB*18] if available. This allows the rasterizer to terminate early-depth testing for a primitive if at least one fragment shader instance was invoked (i.e., one fragment is visible) [S3][S4, unofficial]. Fewer primitives increase the likelihood of this early termination [S3][S4, unofficial].
    *   **Multi-view Rendering:** When rendering per-eye, generate one quad per eye. This is simpler than rendering 3+ sides if a block is located between viewer eyes on cardinal axes [S3][S4, unofficial].

4.  **Fragment Shader: Visibility Buffer Update**
    *   For each fragment passing the early depth test and representative fragment test, write the implicit ID of the block into a preallocated slot of the visibility buffer [S3][S4, unofficial].

5.  **Compute Shader: PVS Generation and Compactification**
    *   Run a bit compactification kernel on the visibility buffer to create indices for $PVS_{curr}$ [S3][S4, unofficial].
    *   Build two PVS sets in a final compute shader pass:
        *   $PVS_{curr}$: Used in the next frame for best occluders based on temporal coherence [S3][S4, unofficial].
        *   $PVS_{curr-prev}$: Determines blocks that became visible since the last frame [S3][S4, unofficial].
    *   Compute shaders are used here to allow larger block sizes and better utilization of streaming multiprocessors compared to mesh shaders [S3][S4, unofficial].

### Parameters
*   **Block Analysis Size:** 256 blocks per task shader workgroup [S4, unofficial].
*   **Mesh Shader Workgroup Size:** 32 blocks [S4, unofficial].
*   **Isovalue:** The scalar threshold for isosurface extraction [S4, unofficial].
*   **Representative Fragment Test:** Enabled via graphics API extension to optimize early depth termination [S3][S4, unofficial].

### Edge Cases and Pitfalls
*   **Conservatism:** The method is conservative, not probabilistic, ensuring no occupied blocks are incorrectly culled [S4, unofficial]. This contrasts with probabilistic methods like [IRR*22] [S4, unofficial].
*   **Multi-view Rendering:** If using 3 visible cube faces instead of a single quad, one must account for rendering more than 3 sides if a block is located between the viewer's eyes on any cardinal axis [S3][S4, unofficial]. Using a single quad per eye simplifies this [S3][S4, unofficial].
*   **Temporal Coherence:** Effective use of occlusion culling historically required careful exploitation of spatial and temporal coherence [S1, S3]. The method maintains $PVS_{curr-prev}$ to handle blocks becoming visible [S3][S4, unofficial].
*   **Precision:** The min-max texture lookup must be precise enough to avoid false positives (blocks marked occupied when they are not) or false negatives (occupied blocks culled) [S4, unofficial].

## Subtopics

### [Occlusion Culling Granularity](1671-occlusion-culling-granularity.md)
Defines the spatial resolution at which visibility is determined, balancing accuracy against computational overhead. It sets the size of culling regions (typically aligned with leaf-level octree nodes) and influences aliasing artifacts.

### [Volumetric Occlusion Culling via Occlusion Queries](4594-volumetric-occlusion-culling-via-occlusion-queries.md)
Determines block visibility by rasterizing bounding boxes and checking depth test passes, incorporating accumulated opacity into the depth test to ensure blocks behind fully opaque foreground volumes are correctly culled.

### [Hierarchical Raster Occlusion Culling](6061-hierarchical-raster-occlusion-culling.md)
A fully GPU-driven technique that reduces rasterization overhead by performing rasterization on a coarser level for group nodes and refining visibility results through ray traversal down to the leaf level. It builds on Raster Occlusion Culling by exploiting spatial coherence through hierarchical grouping.

### Hierarchical Visibility Mask
A visibility culling technique where triangles are marked as visible and forwarded to graphics hardware rather than being rendered by the CPU [S5, book]. It projects polygons onto a hierarchical structure (meta-tiles) and recursively checks subtiles for potential visibility, leading to a logarithmic search [S5, book]. If a meta-cell size is approximately the size of a single pixel, recursion stops, and a single point with an associated normal (estimated by the field gradient) is forwarded to the graphics hardware [S5, book]. This differs from Greene's implementation by not rendering the visible portion on the CPU but relying on the graphics accelerator [S5, book].

### Cluster Visibility Buffer Culling
A culling technique using a visibility buffer to prune non-visible clusters, performing back-face and occlusion culling without requiring a hierarchical Z-buffer or storing cluster bounds/normal cones [S6]. It uses the last metacell of each cluster to store a cluster bound (AABB) [S6]. Visibility is updated via a compute shader pass after cell rasterization, and a second visibility buffer records new clusters becoming visible to trigger full cell rasterization in a second pass [S6]. Pitfalls include flickering artifacts during fast camera movements when non-visible clusters become visible [S6].

## Operating characteristics
*   **Time Complexity:** The culling pass is dominated by the rasterization of proxy geometry. The number of primitives is proportional to the number of potentially occupied blocks. Stream compaction is $O(N)$ where $N$ is the number of blocks [S3][S4, unofficial].
*   **Memory Complexity:** Requires storage for the min-max texture ($O(V)$ where $V$ is volume size), visibility buffer ($O(W \times H)$), and PVS lists ($O(B)$ where $B$ is number of blocks) [S4, unofficial].
*   **Performance:** Outperforms methods that process depth textures of previously established occluder representations, such as hierarchical occlusion maps [ZMHH97] and iterative depth warping [LKE18] [S1, S3]. It is effective for scenes with high depth complexity [S3].
*   **Scalability:** Not scalable for extremely high polygon counts in general scenes if the fragment shader is heavy, but effective for volume rendering where the fragment shader cost is amortized or avoided by culling [S2, unofficial].
*   **Strengths:** Conservative visibility determination (no false culls) [S4, unofficial]; efficient use of GPU rasterization hardware; reduced primitive count (2 triangles per block) improves early depth termination [S3][S4, unofficial].
*   **Limitations:** Requires precomputed min-max textures [S4, unofficial]; relies on rasterization hardware which may have overhead for very small proxy geometries if not batched effectively [S4, unofficial].

## Background
Raster Occlusion Culling was introduced by Kubisch and Tavenrath [KT14] as the state-of-the-art approach for efficient visibility determination in general scenes [S1, S3][S4, unofficial]. It replaced earlier GPU-driven occlusion culling methods that relied on hardware occlusion query objects [CCG*07], which required careful exploitation of spatial and temporal coherence [S1, S3]. Follow-up works include Lee and Lee [LL20] (iterative hierarchical version) and Lee et al. [LJSL21] (fully GPU-driven hierarchical algorithm with ray traversal) [S1, S3]. The method is widely used in direct isosurface rasterization pipelines for scalar volumes [S4, unofficial].

## Sources
[S1] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://doi.org/10.1111/cgf.14670, fetched 2026-10-27
[S2] Voxel Cone Tracing - GitHub, https://raw.githubusercontent.com/righier/gidemo/master/report.pdf, fetched 2026-10-27 (unofficial)
[S3] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://www.uni-weimar.de/en/media/chairs/computer-science-department/vr/research/real-time-rendering/efficient-direct-isosurface-rasterization-of-scalar-volumes/, fetched 2026-11-01
[S4] Efficient Direct Isosurface Rasterization of Scalar Volumes - EG, https://diglib.eg.org/items/8a65e0f0-c048-4fce-9d0b-5acf07452485, fetched 2026-10-30 (unofficial)
[S5] Fast Isosurface Extraction Methods for Large Image Data sets, https://publications.sci.utah.edu/publications/livnat08/Large-scale-isosurface08.pdf, fetched 2026-10-28 (book)
[S6] Clustered Grid Cell Data Structure for Isosurface Rendering, https://doi.org/10.24132/jwscg.2020.28.2, fetched 2026-10-26
