# ExaBricks Hierarchy

Also called: ExaBricks, ExaBricks data structure

## Overview
ExaBricks is a rendering method for Block-Structured Adaptive Mesh Refinement (BS-AMR) data that reorganizes input cells into compact, non-overlapping bricks to enable high-quality smooth interpolation across level boundaries and efficient hardware-accelerated ray tracing [S1]. It solves the problem of costly cell location kernels and discontinuous interpolation at AMR refinement boundaries by decoupling storage (bricks) from spatial traversal (Active Brick Regions) [S1]. Unlike prior methods that require deep tree traversals or stitching unstructured meshes, ExaBricks uses a spatial partitioning of brick support regions to provide $O(1)$ cell retrieval for reconstruction and supports adaptive sampling via an RTX BVH [S1].

## How it works
The method operates using three inter-operating data structures: Bricks, Active Brick Regions (ABRs), and an RTX Bounding Volume Hierarchy (BVH) [S1].

### Inputs and Outputs
**Input:** A set of AMR cells, each defined by lower-left coordinates $(i, j, k)$, level $l$, and a scalar value $v$. Level 0 is the finest level with implicit cell size $1^3$; level $l$ has cell size $2^l$ [S1]. Cells are arranged in Block-Structured or Octree layouts [S1].
**Output:** A rendered image via ray marching, using the ExaBricks structures for traversal and reconstruction [S1].

### Data Structures

**1. Brick**
A brick is a compact, non-overlapping group of same-level cells. The original AMR hierarchy is discarded [S1].
*   **Fields:**
    *   `lower_left`: $(i, j, k)$, the integer coordinates of the brick's lower-left corner.
    *   `level`: $l$, the AMR level of the cells within.
    *   `dims`: $(N, M, K)$, the number of cells in each dimension.
    *   `data`: A 3D array of size $N \times M \times K$ storing the scalar values [S1].

**2. Active Brick Region (ABR)**
An ABR is a leaf node in a spatial partitioning structure. It stores a list of bricks that influence the region's volume [S1].
*   **Fields:**
    *   `bounding_box`: The 3D axis-aligned bounding box of the region.
    *   `active_bricks`: A list of Brick IDs that have non-zero support within this box.
    *   `min_val`: The minimum scalar value of any cell whose support overlaps the region.
    *   `max_val`: The maximum scalar value of any cell whose support overlaps the region.
    *   `finest_cell_width`: The cell width of the finest AMR level influencing the region [S1].

**3. RTX BVH**
A Bounding Volume Hierarchy built over the ABRs. Each node contains the bounding box of its children. Leaves point to ABRs [S1].

### Reconstruction Filter
The method uses the basis method (hat function) for smooth interpolation [S1].
For a cell $C$ of width $C_w$, the basis function at position $p$ is:
$$ \hat{H}_C(p) = \hat{h}\left(\frac{|C_{p_x} - p_x|}{C_w}\right) \hat{h}\left(\frac{|C_{p_y} - p_y|}{C_w}\right) \hat{h}\left(\frac{|C_{p_z} - p_z|}{C_w}\right) $$
where $\hat{h}(x) = \max(1 - x, 0)$ [S1].

The reconstructed value $B(p)$ at position $p$ is:
$$ B(p) = \frac{\sum_{C_i} \hat{H}_{C_i}(p) C_{v_i}}{\sum_{C_i} \hat{H}_{C_i}(p)} $$
where $C_i$ are all cells with non-zero support at $p$, and $C_{v_i}$ is the data value of cell $C_i$ [S1].

The support of a cell $C$ is a rectangular box extending exactly half a cell width beyond the cell itself. The support of a brick is the union of the supports of its cells, resulting in a box exactly half the brick's cell width larger than the brick [S1].

### Algorithm Steps

#### 1. Brick Generation
Reorganize input AMR cells into bricks.
1.  Build a k-d tree top-down from the input cells.
2.  For the current node:
    a.  Check if all cells in the node are on the same AMR level.
    b.  Check if the cells completely fill their combined bounding box.
    c.  If both true, create a leaf node (a Brick).
    d.  If not, split the node along the longest axis of the bounding box.
    e.  Round split positions to an integer multiple of the current coarsest cell width.
3.  Limit leaf size: Bricks must be at most 32 cells wide on any axis.
4.  Discard the tree structure; store only the leaves (Bricks) [S1].

#### 2. ABR Construction
Build a spatial partitioning of brick support regions.
1.  Create a list of "brick support fragments". For each brick, create a fragment consisting of its support box and its Brick ID.
2.  Compute the global bounding box of all fragments.
3.  Recursively subdivide the global bounding box:
    a.  For the current region and its overlapping fragments:
    b.  Find a partitioning plane from the fragment boundaries that is closest to the region's spatial center. Prefer the widest dimension.
    c.  If no valid plane exists (e.g., only one fragment overlaps or boundaries are coincident), create a leaf ABR containing the current list of brick IDs.
    d.  Else, partition the region into left and right sub-regions. Sort fragments into left/right subtrees based on overlap.
    e.  Recurse on sub-regions.
4.  For each resulting leaf ABR:
    a.  Store the list of active brick IDs.
    b.  Compute and store the bounding box.
    c.  Compute `min_val` and `max_val` from the scalar values of all cells in the active bricks whose support overlaps the region.
    d.  Compute `finest_cell_width` from the levels of the active bricks [S1].

#### 3. BVH Construction
Build an RTX BVH over the ABRs.
1.  Insert all ABRs as leaves into a BVH.
2.  Compute bounding boxes for internal nodes as the union of their children's boxes [S1].

#### 4. Ray Marching and Reconstruction
1.  Trace a ray through the RTX BVH.
2.  For each leaf ABR hit:
    a.  Iterate over the `active_bricks` list.
    b.  For each brick, iterate over its cells to find those with non-zero support at the current ray sample position $p$.
    c.  Since bricks are 3D grids, cell indices can be computed trivially from $p$ and the brick's `lower_left` and `level`.
    d.  Accumulate $\hat{H}_{C_i}(p) C_{v_i}$ and $\hat{H}_{C_i}(p)$ for the reconstruction formula.
3.  Use `min_val` and `max_val` of the ABR for empty space skipping.
4.  Use `finest_cell_width` for adaptive sampling [S1].

### Parameters
*   **Brick Size:** Bricks can be small (1, 2, or 4 cells) at level boundaries, but larger in homogeneous regions. Max width is 32 cells [S1].
*   **Level Definition:** Level 0 is finest ($1^3$). Level $l$ is $2^l$. Coordinates are lower-left unit coordinates [S1].

### Edge Cases and Pitfalls
*   **Precision:** Split positions in brick generation must be rounded to integer multiples of the coarsest cell width to ensure alignment [S1].
*   **Boundary Handling:** The basis function support extends beyond the brick, requiring ABRs to handle overlapping supports from different levels. Nearest-neighbor filters do not require ABRs, but smooth filters do [S1].

## Subtopics

### [Octant Method](2648-octant-method.md)
A reconstruction strategy for BS-AMR data that maps sample points to scalar values by decomposing the domain into non-overlapping rectilinear octants. It ensures $C^0$ continuity across level boundaries and supports fast implicit ray-isosurface intersections. Unlike ExaBricks, which uses bricks and ABRs for smooth interpolation, the Octant Method uses octants that do not cross level boundaries and tile the domain without gaps.

## Operating characteristics
*   **Time Complexity:** Brick generation is $O(N \log N)$ due to k-d tree construction. ABR construction is recursive and depends on the complexity of support overlaps. Ray traversal is accelerated by the RTX BVH, providing $O(\log M)$ traversal over ABRs, where $M$ is the number of ABRs. Reconstruction is $O(K)$ per sample, where $K$ is the number of active bricks in the current ABR leaf [S1].
*   **Memory Complexity:** Stores bricks (compact same-level cells) and ABRs. The ABR structure adds overhead compared to simple brick lists but avoids per-sample k-d tree traversals [S1].
*   **Performance:** Avoids costly cell location kernels required by prior methods (e.g., Wald et al. [37]). Enables hardware-accelerated ray marching and adaptive sampling [S1].
*   **Strengths:** High-quality smooth interpolation across AMR boundaries; efficient space skipping and adaptive sampling; no need for deep tree traversals during rendering [S1].
*   **Limitations:** Requires pre-processing to build bricks and ABRs; support regions for smooth filters can be large, leading to many active bricks per ABR [S1].

## Background
ExaBricks was introduced in 2020 by Wald et al. [S1]. It builds on the brick organization method by Kähler et al. [17] and the basis reconstruction method by Wald et al. [37]. The method addresses the limitations of prior AMR rendering approaches that suffered from discontinuous interpolation or high computational cost for cell location [S1].

## Sources
[S1] Ray Tracing Structured AMR Data Using ExaBricks, https://www.sci.utah.edu/~wald/Publications/2020/exabrick/exabrick.pdf, fetched 2026-10-30
