# Ray-grid traversal

Also called: Voxel traversal, Grid traversal, DDA traversal, Amanatides-Woo traversal

## Overview
Ray-grid traversal computes the ordered sequence of grid cells (voxels) intersected by a ray, returning the cell indices along with entry and exit distances (or times) for each segment [S1, S2, unofficial][S5]. It solves the problem of efficiently identifying which volumetric data elements a ray passes through without performing expensive per-voxel bounding box intersection tests or fixed-distance sampling [S5][S7, unofficial]. It sets itself apart from [Direct volume ray casting](19-direct-volume-ray-casting.md) by providing the geometric traversal logic (the "where" and "when") rather than the shading/compositing logic (the "what"), and from [Acceleration](18-acceleration.md) by operating on uniform or structured grids rather than hierarchical spatial partitioning structures like octrees [S5, S2451].

## Shared principles
All ray-grid traversal methods share the core mechanism of advancing analytically from one cell boundary to the next, avoiding fixed-distance sampling steps [S5][S7, unofficial]. The algorithm determines the next cell by comparing the distances to the upcoming boundaries on the x, y, and z axes, selecting the minimum distance to advance [S2, S3, unofficial][S5].

### Mathematical Foundation
Let the ray be defined by origin $\mathbf{o}$ and direction $\mathbf{d}$ (normalized or unnormalized). The grid is defined by cell boundaries. For a uniform grid with voxel size $v$ and dimensions $N_x, N_y, N_z$, the traversal relies on three key precomputed values per axis $i \in \{x, y, z\}$ [S2, S3, unofficial][S5]:

1.  **Step Direction ($\text{step}_i$):** Determines the index increment direction.
    $$
    \text{step}_i = \begin{cases} 
    +1 & \text{if } d_i > 0 \\
    -1 & \text{if } d_i < 0 \\
    0 & \text{if } d_i = 0 
    \end{cases}
    $$
    If $d_i = 0$, the ray never crosses a boundary in that dimension, and the corresponding distance is treated as infinity [S5].

2.  **Time/Distance Delta ($\Delta t_i$ or $s_{\Delta i}$):** The distance along the ray to traverse one voxel width.
    $$
    \Delta t_i = \frac{v_i}{|d_i|} \cdot \text{step}_i \quad \text{or} \quad s_{\Delta i} = \frac{\Delta x_i}{|\widehat{d}_i|}
    $$
    where $v_i$ is the voxel size and $\widehat{d}_i$ is the unit direction component [S2, unofficial][S5].

3.  **Time/Distance to Next Boundary ($t_{max, i}$ or $s_{max, i}$):** The distance from the ray origin to the first boundary crossed in axis $i$.
    $$
    t_{max, i} = \frac{\text{boundary}_i - o_i}{d_i}
    $$
    The initial boundary is determined by the starting voxel index. If the ray starts inside the grid, the initial boundary is the next one in the direction of travel [S2, S3, unofficial][S5].

### Traversal Loop
The core loop iterates while the ray remains within the grid bounds [S5]:
1.  Identify the axis $k$ with the minimum boundary distance: $t_{min} = \min(t_{max, x}, t_{max, y}, t_{max, z})$.
2.  Record the current cell and the interval $[t_{entry}, t_{min}]$.
3.  Advance the index in axis $k$: $\text{index}_k \leftarrow \text{index}_k + \text{step}_k$.
4.  Update the boundary distance for axis $k$: $t_{max, k} \leftarrow t_{max, k} + \Delta t_k$.
5.  Update $t_{entry} \leftarrow t_{min}$.

### Edge and Corner Handling
When the ray crosses an edge or corner (i.e., two or three $t_{max}$ values are equal), the traversal must advance all corresponding axes to maintain 4-connectivity (or 6/8-connectivity in 2D/3D) and avoid zero-length segments [S5, S3436]. The implementation uses independent `if` checks for ties rather than `if/elif` chains [S5]. A tie tolerance $\varepsilon_{\mathrm{DDA}}$ is used to handle floating-point drift, typically defined as $64 \varepsilon_{\mathrm{machine}} \max(1 \text{ cm}, L_{\mathrm{grid}})$ [S5].

### Non-Uniform Grids
For non-uniform grids defined by arbitrary 1D edge arrays (walls) $W_i$, the traversal time to the next wall is computed using the inverse velocity [S1, unofficial]:
$$
t_{hit} = (W_{next} - o) \cdot \frac{1}{d}
$$
The traversal state maintains `hittimes` for each dimension, updated by precomputed `voxel_traversal_times` which depend on the wall spacing and ray velocity [S1, unofficial].

## Subtopics

### [Hierarchical Trilinear Ray Casting](851-hierarchical-trilinear-ray-casting.md)
A method that accelerates ray casting by traversing a multi-dimensional hierarchy (mdh) and adaptively selecting the resolution level for data evaluation based on a user-defined error tolerance and data importance weighting. It differs from siblings like [Hierarchical Digital Differential Analyzer (HDDA)](1122-hierarchical-digital-differential-analyzer-hdda.md) by using least-squares trilinear basis functions to model data regions, allowing the algorithm to decide whether to refine traversal or evaluate the region directly based on the approximation error. Unlike methods that rely on simple homogeneity, this approach generalizes homogeneity to least-squares fits.

### [Hierarchical Digital Differential Analyzer (HDDA)](1122-hierarchical-digital-differential-analyzer-hdda.md)
A specialized ray traversal algorithm for sparse voxel grids (specifically VDB structures) that performs multi-resolution leapfrogging to efficiently determine intersected voxels or tiles. It solves the problem of slow ray traversal in deep, sparse hierarchies by leveraging the fixed compile-time tree configuration to inline and collapse vertical navigation overhead. Unlike standard DDA, HDDA leaps across empty space at multiple resolution levels.

### [Front-to-Back Layer Traversal](1212-front-to-back-layer-traversal.md)
A rendering order strategy for Matryoshka layers that processes cells in front-to-back order along the viewing direction to handle occlusion correctly. It solves the problem of correctly rendering nested volumetric layers by ensuring that outer cells occlude inner cells. It sets itself apart from standard ray casting by operating on discrete layer planes rather than continuous ray steps.

### [Dual Grid Traversal](1233-dual-grid-traversal.md)
Constructs a secondary grid structure (the dual grid) from a primary voxel grid (the primal grid) to facilitate isosurface extraction or topological analysis. It solves the problem of mapping discrete voxel data to a continuous vertex-based representation by placing a vertex at the center of each primal cube. It sets itself apart from standard marching cubes by operating on the dual lattice, where each dual cell spans a $3 \times 3 \times 3$ region of the primal grid.

### [Voxel Ray Marching with Traversal Order Optimization](1251-voxel-ray-marching-with-traversal-order-optimization.md)
Renders a 3D voxel texture by marching rays from the camera through the volume, compositing samples using front-to-back alpha blending. It differs from its parent [Ray-grid traversal](17-ray-grid-traversal.md) by explicitly optimizing the traversal order of voxel boundaries using precomputed time deltas ($t_{\Delta}$) and maximum boundary times ($t_{max}$), minimizing floating-point operations during the loop.

### [Backward Projection](1491-backward-projection.md)
Maps volumetric data attributes from the 3D voxel grid onto a 2D image plane by accumulating contributions along rays. It solves the problem of visualizing volumetric features by aggregating weighted voxel values into image pixels. Unlike forward projection methods, backward projection traverses the volume for each pixel, allowing for flexible compositing and transfer function application.

### [Sparse Voxel Octree Ray Traversal](2451-sparse-voxel-octree-ray-traversal.md)
A hierarchical ray-casting algorithm that traverses a 3D sparse voxel octree to find the first occupied leaf node (voxel) intersected by a ray. It solves the problem of efficiently rendering large, sparse 3D volumes by skipping empty space and adapting resolution based on viewer distance (implicit Level-of-Detail). It sets itself apart from sibling methods by using a fixed-size traversal stack or iterative descent to manage backtracking through the octree hierarchy.

### [Ray Marching with Octant Masking](2488-ray-marching-with-octant-masking.md)
Traverses a uniform voxel grid by advancing a ray in discrete steps and using an octant mask to determine visibility or traversal direction based on the ray's direction vector components. It solves the problem of efficiently stepping through dense volume data without hierarchical acceleration structures, relying instead on simple arithmetic and bitwise operations to manage the traversal state.

### [Hierarchical Grid Traversal](2560-hierarchical-grid-traversal.md)
Accelerates volume rendering by stepping through a multi-resolution grid hierarchy (mipmap) rather than a single flat grid. It starts at the root level and recursively visits child voxels only if the ray intersects the parent voxel's axis-aligned bounding box (AABB), effectively skipping empty space at coarse levels before refining the search at finer levels. Unlike flat [Digital Differential Analyzer (DDA) Traversal](3061-digital-differential-analyzer-dda-traversal.md) which samples every voxel along the ray path, this method reduces sample evaluations.

### [Image-order Rendering Loop](2938-image-order-rendering-loop.md)
A rendering loop that iterates over pixels, generates rays based on camera parameters (perspective or orthographic), and calls the raymarcher to shade the scene.

### [Coherent Grid Traversal](2987-coherent-grid-traversal.md)
A ray tracing acceleration technique for uniform grids that traverses bundles (frustums) of coherent rays slice-by-slice along a dominant axis, abandoning individual cell stepping to avoid state invalidation caused by ray divergence within a packet. It sets itself apart from [Digital Differential Analyzer (DDA) Traversal](3061-digital-differential-analyzer-dda-traversal.md) by processing entire grid slices as a coherent whole, allowing SIMD acceleration and avoiding the packet-splitting overhead required when rays diverge.

### [Digital Differential Analyzer (DDA) Traversal](3061-digital-differential-analyzer-dda-traversal.md)
The foundational algorithm for ray-grid traversal, introduced by Amanatides and Woo in 1987. It advances analytically from one cell boundary to the next, maintaining state variables for the current voxel index and the distance to the next boundary on each axis. It is the basis for most uniform grid traversal implementations.

### [Z-order curve traversal with run-length encoding](3408-z-order-curve-traversal-with-run-length-encoding.md)
Not given in the sources.

### [Hero-like 4-Connectivity (H4T)](3436-hero-like-4-connectivity-h4t.md)
A ray traversal method that uses a 2x2 cell test to determine the first cell the ray enters, ensuring 4-connectivity. It uses a stable sorting algorithm on t-values and axis identifiers to resolve ties.

### [SVO Ray Tracing](3451-svo-ray-tracing.md)
Not given in the sources.

### [GPU CSG Ray Tracing with Uniform Grid Index](4098-gpu-csg-ray-tracing-with-uniform-grid-index.md)
A GPU-based ray tracer for Constructive Solid Geometry (CSG) that uses a uniform grid index over the root universe's cells to accelerate ray traversal, replacing the unindexed full-scan approach.

### [Octree and Tree Traversal](4134-octree-and-tree-traversal.md)
Covers methods for traversing hierarchical spatial partitioning structures, primarily octrees, to accelerate ray casting in volume rendering. By skipping empty or homogeneous regions, these methods reduce the number of memory accesses and shading computations compared to uniform grid traversal. They differ from siblings like [Hierarchical Trilinear Ray Casting](851-hierarchical-trilinear-ray-casting.md) and [Hierarchical Digital Differential Analyzer (HDDA)](1122-hierarchical-digital-differential-analyzer-hdda.md) by explicitly managing the tree structure's branching logic and node pointers.

### [Multi-Grid Ray Traversal](4142-multi-grid-ray-traversal.md)
Not given in the sources.

### [Wide DDA Cell Traversal](5122-wide-dda-cell-traversal.md)
An algorithm to traverse a 4x4x4 grid of 'wide cells' within an AABB using a DDA (Digital Differential Analyzer) approach, generating bounding boxes and t-intervals for each cell to optimize octree traversal.

### [Stackless Traversal](5202-stackless-traversal.md)
A method for traversing hierarchical volume structures (specifically N3-trees) that eliminates the need for a stack data structure by recalculating Digital Differential Analyzer (DDA) parameters at each tree descent and ascent. It differs from sibling methods like [Leaf Collection](899-leaf-collection.md) and [Point Plane Test Traversal](982-point-plane-test-traversal.md) by relying on parent pointers for ascent and full DDA recalculation rather than incremental updates or explicit leaf enumeration, optimizing for GPU register usage and branch divergence.

### Barycentric Coordinate Traversal
An alternative traversal algorithm that uses barycentric coordinates (u, v, w, x) to determine the child tetrahedron, avoiding explicit plane tests [S9, unofficial]. It calculates barycentric coordinates for the ray point within the current tetrahedron and uses boolean logic on their sums to select the child tetrahedron in an adaptive mesh refinement (AMR) hierarchy [S9, unofficial].

### Perspective-Correct AMR Patch Traversal
A hybrid CPU-GPU traversal strategy where the CPU traverses the KD-tree of AMR patches back-to-front, and the GPU traverses cells within a patch. Sub-patches are generated to ensure perspective-correct sorting of cells relative to the observer [S10].

### Lazy Sweep Ray Casting
A fast algorithm for rendering general irregular grids based on the sweep-plane paradigm, accelerating ray casting by exploiting spatial coherence and handling disconnected/nonconvex grids [S11, unofficial]. It is substantially faster (by up to two orders of magnitude) than other algorithms implemented in software [S11, unofficial].

### Majorant Supergrid Traversal
A medium traversal structure using a supergrid to allow empty and thin regions to be crossed in far fewer null collisions than paying for the densest voxel in the volume [S12, unofficial]. It consumes null collisions in one fused C++ walk instead of iterating them in Python, sharing a single loop with the DDA traversal when a supergrid is used [S12, unofficial].

### Absolute Position Calculation
Calculates the world-space position of a node's corner by summing scaled unit vectors from the traversal stack [S13, unofficial].

### WebGPU Renderer Optimization
Evaluates each visible channel at each sample during ray traversal to reduce diverging branches. Starts tree traversal at a user-defined subdivision level, or the parent level of the subdivision level where traversal was terminated for the previous sample [S14].

### Grid Traversal Setup (CUDA)
A CUDA device function that initializes the parameters for stepping through a voxel grid along a ray [S7, unofficial].

### Ray Connectivity
A property defining the set of cells a ray can traverse to from its current cell, inspired by pixel connectivity in image processing. In 2D, there are 4-connectivity (orthogonal neighbors) and 8-connectivity (including diagonal neighbors) [S15, unofficial].

### Block Walking
An algorithm that traverses all blocks a ray intersects with, starting from closest to the camera, to measure the direct impact of block-walking on performance by writing to a scalar at each block visit [S16, unofficial]. Block walking reduces the number of queries (acceleration structure accesses) by 5-10x compared to standard ray tracing, translating to a 1.5x average performance improvement in block traversal mode [S16, unofficial].

### Adaptive grid traversal dispatcher
A dispatcher function that selects the appropriate traversal variant based on the maximum range of the segment. If maxRange < 256.0f, it calls `traverseGridSegment16` (Q8.8 fixed-point); otherwise, it calls `traverseGridSegment32` (Q24.8 fixed-point) [S17, unofficial].

### Indicator Grid Ray Marching
A binary map cached and updated during training to signify empty areas, allowing for early termination of ray marching based on transmittance, thereby optimizing ray marching efficiency [S18].

## Background
The foundational algorithm for uniform grid ray traversal was introduced by J. Amanatides and A. Woo in "A Fast Voxel Traversal Algorithm for Ray Tracing," published in Eurographics '87 [S2, unofficial]. This algorithm, often referred to as the Amanatides-Woo algorithm or 3D DDA, revolutionized ray tracing by providing an efficient way to traverse voxel grids without per-voxel intersection tests [S2, S6, unofficial]. It has since become the standard for uniform grid traversal in volume rendering and voxel-based rendering [S3, S6, unofficial].

Subsequent developments have extended this core logic to handle non-uniform grids [S1, unofficial], hierarchical structures like octrees and VDBs [S4, S1122], and GPU-optimized variants for coherent ray packets [S2987]. The algorithm is widely used in real-time voxel rendering (e.g., Minecraft-style games) [S6, unofficial], scientific visualization [S1, unofficial], and neural radiance field (NeRF) rendering [S7, unofficial].

## Sources
[S1] jw3126/VoxelRayTracers.jl, https://github.com/jw3126/VoxelRayTracers.jl, fetched 2026-10-24 (unofficial)
[S2] francisengelmann/fast_voxel_traversal, https://github.com/francisengelmann/fast_voxel_traversal, fetched 2026-10-24 (unofficial)
[S3] Victorique-GOSICK/Voxel-3D-DDA, https://github.com/Victorique-GOSICK/Voxel-3D-DDA, fetched 2026-10-24 (unofficial)
[S4] Super Fast Ray Casting in Tiled Worlds using DDA, https://www.youtube.com/watch?v=NbSee-XM7WA, fetched 2026-10-25 (unofficial)
[S5] 2026_surface_mesh_tools/docs/collision_detection/04_ray_grid_traversal ..., https://github.com/canesesmarjf/2026_surface_mesh_tools/blob/main/docs/collision_detection/04_ray_grid_traversal.md, fetched 2026-10-25
[S6] Minecraft Voxel DDA Raycast - Shadertoy, https://www.shadertoy.com/view/WccyWl, fetched 2026-10-26 (unofficial)
[S7] nerfstudio-project/nerfacc, https://github.com/nerfstudio-project/nerfacc, fetched 2026-10-26 (unofficial)
[S8] viljanvr/VisualNifty, https://github.com/viljanvr/VisualNifty, fetched 2026-10-26 (unofficial)
[S9] Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data, https://repository.bilkent.edu.tr/items/fcbc592a-2646-49af-8b84-67308b6fd3e9, fetched 2026-10-27 (unofficial)
[S10] High-Quality, Semi-Analytical Volume Rendering for AMR ..., https://people.freedesktop.org/~marcheu/vis09-amr.pdf, fetched 2026-10-26
[S11] Volume and Isosurface Rendering with GPU‐Accelerated Cell Projection, https://www.researchgate.net/publication/227523087_Volume_and_Isosurface_Rendering_with_GPU-Accelerated_Cell_Projection, fetched 2026-10-27 (unofficial)
[S12] Sample matching for extinction gradients (review branch), https://github.com/AuroraRyan0301/mitsuba3/pull/1, fetched 2026-10-26 (unofficial)
[S13] realazthat/landscapes, https://github.com/realazthat/landscapes, fetched 2026-10-26 (unofficial)
[S14] Residency Octree: A Hybrid Approach for Scalable Web ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC10840607/, fetched 2026-10-29
[S15] versal algorithm for ray tracing Hur man gör en hierarkisk ..., https://kth.diva-portal.org/smash/get/diva2:2069409/FULLTEXT01.pdf, fetched 2026-10-25 (unofficial)
[S16] [PDF] Sparse Volume Rendering using Hardware Ray Tracing and Block ..., https://gpuopen.com/download/SA2021_BlockWalk.pdf, fetched 2026-10-26 (unofficial)
[S17] FastLED: fl/traverse_grid.h Source File, https://fastled.io/docs/d9/d42/traverse__grid_8h_source.html, fetched 2026-10-30 (unofficial)
[S18] Efficient Deformable Tissue Reconstruction via Orthogonal Neural Plane, https://doi.org/10.1109/tmi.2024.3388559, fetched 2026-10-26
