# Parallel K-d Tree Construction

Also called: Cooperative Spatial Partitioning, GPU k-d tree, Parallel K-d Tree Spatial Partitioning, Parallel Space Partitioning for Volume Rendering, ZSL18/ZSL19 k-d tree, ZSL19 GPU k-d tree

## Overview
Parallel k-d tree construction algorithms partition sparse volumetric or unstructured data into spatial regions to enable efficient ray traversal, early termination, and workload distribution in parallel volume rendering [S1, unofficial][S4, unofficial][S5]. Unlike static partitioning, these methods dynamically determine split planes using occupancy queries or median selection to balance tree depth and node count [S1, unofficial][S5]. They differ from [Real-time KD Tree Construction](4437-real-time-kd-tree-construction.md) by focusing on distributed-memory or GPU-accelerated top-down recursion rather than single-node heuristic optimization, and from [K-D Tree Subdivision](8148-k-d-tree-subdivision.md) by explicitly managing inter-processor communication or GPU kernel synchronization during the build phase [S1, unofficial][S3].

## How it works
The method varies based on the target architecture (CPU multi-core, GPU, or MPI distributed) [S1, unofficial][S3][S4, unofficial]. The core logic involves decomposing the volume, determining split planes via occupancy or median finding, and recursively partitioning data [S1, unofficial][S5].

### Inputs and Outputs
*   **Inputs:** Volumetric data (voxels or cells), transfer function (for occupancy), number of processors/threads, halting criteria (target tree height or node count) [S1, unofficial][S4, unofficial].
*   **Outputs:** A k-d tree structure (nodes with bounding boxes and split planes), optionally a serialized file (e.g., LODTree) or in-memory hierarchy [S3][S4, unofficial].

### Data Structures
#### Summed Volume Table (SVT) [S1, unofficial]
Used in CPU/GPU hybrid approaches to query occupancy [S1, unofficial].
*   **Layout:** 3D integer array matching brick dimensions ($32^3$) [S1, unofficial].
*   **Content:** `SVT[x][y][z]` stores the sum of non-empty voxels in the box from $(0,0,0)$ to $(x,y,z)$ [S1, unofficial].
*   **Bit Packing:** Not specified; typically standard integer types [S1, unofficial].

#### LODTree File Structure [S4, unofficial]
Used in MPI-based LODTree construction [S4, unofficial].
*   **Header (`lod-file-header-struct`):**
    *   `signature`: `char[1]`
    *   `source`: `char[7]`
    *   `width, height, depth`: `int`
    *   `num-datagoints`: `int`
    *   `min, max`: `float`
    *   `size`: `int` (Number of LOD nodes)
    *   `resolutions`: `int`
*   **Min-Max Vertex Coords (`char`):**
    *   Format: `XlllXlll / mxyzaxyz`
    *   First 4 bits: Canonical vertex coordinate of cell vertex with minimum data value [S4, unofficial].
    *   Last 4 bits: Canonical vertex coordinate of cell vertex with max value [S4, unofficial].
*   **Cell Dimensions (`char` + `int`):**
    *   First 3 bits: Flags for irregularity in x, y, z [S4, unofficial].
    *   Last 4 bytes: Regular size of cell (max $2^4$) [S4, unofficial].
    *   Format: `Xlllllll / xyzssss`

#### Cell Data [S4, unofficial][S5]
*   **Centroids:** 3D coordinates $(x, y, z)$ for each cell, used for deterministic region placement [S5].

### Algorithm Steps
The implementation depends on the specific variant [S1, unofficial][S3][S4, unofficial].

#### Variant A: CPU/GPU Hybrid with Summed Area Tables [S1, unofficial]
This method uses the CPU for top-down recursion and plane sweeping, offloading tight bounding box computation to the GPU [S1, unofficial].
1.  **Decomposition:** Decompose the volume into bricks of size $32^3$ voxels [S1, unofficial].
2.  **Preclassification:** Generate a binary flag for each voxel indicating if it is empty or non-empty based on the current transfer function [S1, unofficial].
3.  **SVT Construction:** Build a 3D Summed Volume Table (SVT) for each brick in parallel [S1, unofficial]. Brick size $32^3$ ensures the SVT fits in the L1 cache of x64 CPUs [S1, unofficial].
4.  **Root Bounding Box:** Determine the tight bounding box for the root node by shrinking the global AABB to fit non-empty voxels [S1, unofficial]. Perform parallel occupancy queries on SVTs overlapping the global box [S1, unofficial]. Combine local bounding boxes from SVTs to form the global tight box [S1, unofficial].
5.  **Recursive Splitting (Top-Down):**
    *   **Plane Selection:** Use a cost function to find the optimal splitting plane by sweeping candidate planes and inspecting the volume of the bounding boxes to the left and right [S1, unofficial].
    *   **GPU Offload:** For each candidate plane, start two GPU kernels (left and right half-spaces) [S1, unofficial]. Use a binning approach to reduce kernel calls [S1, unofficial]. Align bin boundaries with SVT macro cells ($8^3$) to avoid splitting SVTs [S1, unofficial].
    *   **Bounding Box Computation:** Compute tight bounding boxes for the resulting subregions using parallel reduction (e.g., Thrust) on the GPU [S1, unofficial]. Check if macro cells are within the volumetric region of interest using Morton curve ordering [S1, unofficial].
    *   **Recursion:** Recursively apply steps 4-5 to the left and right children [S1, unofficial].
6.  **Halting:** Stop when the targeted tree height or node count is reached [S1, unofficial].

#### Variant B: Distributed MPI Partitioning [S3]
This method uses a recursive "farm-out" strategy across an MPI group [S3].
1.  **Initialization:** Processor 0 initializes MPI, reads the file, and calculates min/max data values for all voxel cells [S4, unofficial].
2.  **Median Finding:** Processor 0 finds the median cell value to partition the data [S4, unofficial].
3.  **Partitioning:** Processor 0 schedules the top half of the data array to the next processor in the group [S4, unofficial]. Processor 0 recursively partitions the bottom half locally [S4, unofficial]. Each subsequent processor, upon receiving a message, performs the same partitioning: farms out the top half to the next processor (deterministically predicted by a recursive routine) and partitions the bottom half locally [S4, unofficial].
4.  **Message Content:** Messages contain only min/max values, Level-of-Detail (LOD), and an index into the full data array on Processor 0 [S4, unofficial]. This minimizes communication overhead [S4, unofficial].
5.  **Termination:** Subcontracting continues until the limit of the processor array is reached [S4, unofficial]. Remaining processors perform recursive partitioning locally [S4, unofficial].
6.  **Reconstruction:** Processor 0 interrogates each process in the predicted order to retrieve message stubs [S4, unofficial]. The stub order is used to write out the full k-d tree data [S4, unofficial].

#### Variant C: Parallel Recursive Coordinate Bisection [S5]
Used for unstructured data partitioning in ParaView [S5].
1.  **Centroid Calculation:** Find all centroids for all cells to deterministically place each cell in a region [S5].
2.  **Axis Selection:** Choose the coordinate direction to split the region, typically the longest axis, to prevent elongated regions and minimize plane-cell intersections [S5].
3.  **Median Selection:** Find the index of the median cell such that an equal number of cells lie on either side [S5]. Use the Floyd-Rivest selection algorithm, parallelized with prefetching for off-processor tests [S5].
4.  **Partitioning:** Partition cells based on the computed median [S5].
5.  **Recursion:** Apply steps 2-4 recursively until the desired number of regions is reached [S5].

### Parameters
*   **Brick Size:** $32^3$ voxels (CPU/SVT variant) [S1, unofficial].
*   **Macro Cell Size:** $8^3$ (GPU shared memory variant) [S1, unofficial].
*   **Targeted Size:** Height or number of nodes, influencing construction time and rendering performance [S1, unofficial].
*   **Halting Criteria:** Parameters favoring shallow or deep trees [S1, unofficial].
*   **Virtual Voxel Size Increment:** Default 1, for LODTree [S4, unofficial].
*   **Virtual Voxel Overlap Factor:** Prevents small voxels at grid edges [S4, unofficial].
*   **Number of Processors:** Ideally a power of two for efficiency [S4, unofficial].

### Edge Cases and Pitfalls
*   **Identical Values:** The Floyd-Rivest algorithm exhibits worst-case behavior with long sequences of identical values (common in aligned volume models), potentially increasing runtime from seconds to hours [S5]. A modified algorithm to detect and skip duplicates is required [S5].
*   **Memory Limits:** In MPI variants, Processor 0 must fit all simulation data in memory to create the initial array, limiting file size by available memory on the initial processor [S4, unofficial].
*   **Tree Balance:** The implementation does not guarantee a well-balanced tree, but the extra searching overhead is insignificant compared to rendering calculation time [S3].
*   **Java Compatibility:** Requires careful consideration of compiler alignment, limiting maximum array size to a 32-bit signed integer, and defining all numeric fields as signed [S4, unofficial].
*   **Load Imbalance:** If data cells are distributed as connected components, zooming in on a local region causes severe load imbalances because only a few processors handle rendering while others are idle [S2, unofficial].

## Subtopics

### [KDTreeSamplingPartition](8210-kdtreesamplingpartition.md)
KDTreeSamplingPartition is a domain decomposition algorithm for distributing volume rendering samples across multiple compute blocks in a distributed-memory system. It builds a k-d tree structure over spatial data points by recursively splitting blocks along coordinate dimensions using a sampling-based approach to determine split positions. It sets itself apart from static partitioning methods by dynamically determining split planes via random sampling and managing neighbor links via a reduce proxy to handle wrap-around or boundary conditions.

## Operating characteristics
### Time and Memory Complexity
*   **CPU/GPU Hybrid:** Construction time is dominated by GPU kernel launches for bounding box computation and CPU recursion [S1, unofficial]. The binning approach reduces kernel calls [S1, unofficial]. Memory usage is optimized by fitting SVTs into L1 cache ($32^3$) and using shared memory for macro cells ($8^3$) [S1, unofficial].
*   **MPI Partitioning:** The recursive partitioning is the most time-consuming part (at least 90% of serial run time) [S4, unofficial]. Parallelization focuses on this step. Memory is limited by the capacity of Processor 0 to hold the full dataset [S4, unofficial].
*   **Coordinate Bisection:** Selection is sub-linear on average ($O(n)$ expected), but worst-case $O(n^2)$ with duplicate values [S5].

### Measured Performance
*   Specific benchmarks are not provided in the sources, but the GPU variant achieves good utilization due to parallel reductions and smaller macro cell sizes [S1, unofficial].
*   The MPI variant is most efficient when the number of processors is a power of two [S4, unofficial].
*   In a test distributing 1 million cells to 128 processors, about 640,000 bytes of data were transferred to and stored at each processor in the worst case of not sharing node information [S2, unofficial].

### Strengths and Limitations
*   **Strengths:** Adaptive partitioning along orthogonal planes; supports early ray merging and culling; efficient for sparse volumes via occupancy queries; scalable to distributed memory [S1, unofficial][S2, unofficial][S3].
*   **Limitations:** Does not guarantee balanced trees; sensitive to data distribution (duplicate values); high memory requirement for root processor in MPI; complex synchronization in GPU hybrid approach [S1, unofficial][S2, unofficial][S3][S4, unofficial].

### When to Use
*   Use for sparse volume rendering where empty space skipping is critical [S1, unofficial].
*   Use for parallel distributed rendering where spatial locality improves ray merging [S2, unofficial].
*   Use for unstructured data partitioning in visualization tools like ParaView [S5].
*   K-d trees are preferred over octrees because they support adaptive partitioning along orthogonal coordinate planes and allow straightforward determination of the depth ordering of the resulting regions [S3].

## Background
*   **Origin:** The multi-core CPU variant was first presented in [ZSL18] and refined in [ZSL19], loosely based on work by Vidal et al. [VMD08] using summed area tables [S1, unofficial].
*   **MPI Variant:** Developed for the DYNAISOLOD algorithm, targeting SGI and Intel teraflop machines, using a recursive "farm-out" strategy [S4, unofficial].
*   **Coordinate Bisection:** Used in ParaView for parallel unstructured volume rendering, employing the Floyd-Rivest algorithm for median selection [S5].
*   **History:** The methods evolved to address the need for efficient spatial partitioning in large-scale volume rendering, moving from single-node heuristics to distributed and GPU-accelerated constructions [S1, unofficial][S2, unofficial][S3][S4, unofficial].

## Sources
[S1] Comparing Hierarchical Data Structures for Sparse Volume Rendering with Empty Space Skipping, https://doi.org/10.36227/techrxiv.11416422.v1, fetched 2026-10-24 (unofficial)
[S2] Parallel Software Rendering for Large-Scale Volume Visualization, https://escholarship.org/uc/item/489126f2, fetched 2026-10-27 (unofficial)
[S3] A Scalable Parallel Cell-Projection Volume Rendering Algorithm for ..., https://web.cs.ucdavis.edu/~ma/papers/PRS97.pdf, fetched 2026-10-27
[S4] Dynamic isosurface extraction and level-of-detail in voxel space, https://digital.library.unt.edu/ark:/67531/metadc707496/?q=marching cubes, fetched 2026-10-28 (unofficial)
[S5] Parallel Unstructured Volume Rendering in ParaView, https://www.kennethmoreland.com/documents/ParallelVolRen.pdf, fetched 2026-10-27
