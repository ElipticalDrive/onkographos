# Brick Type Selection

Also called: Choosing Optimal Brick Types, Hybrid Adaptive Bricking, Hybrid Bricking, Hybrid Bricking Construction, Hybrid Bricking with Elemental Brick Types, Hybrid Volume Bricking

## Overview
Brick Type Selection is a method for constructing a memory-efficient volumetric data structure by splitting a volume into constant-sized bricks and independently assigning one of four elemental representations (Empty, Dense, Voxel List, or Octree) to each brick based on local sparsity [S1, unofficial]. Unlike conventional bricking, which only distinguishes between empty and non-empty regions (storing full dense data for any non-empty brick), this approach adapts to varying sparsity levels, significantly reducing memory footprint while maintaining high-performance access [S1, unofficial]. It serves as the foundational construction phase for hybrid bricking systems, preceding performance optimization strategies like JiTTree [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset (3D grid of values), a brick side-length $m$ (power-of-two), and a data type size $d$ (bytes).
*   **Output:** A global root array containing tuples of `(type, pointer)` for each brick, and memory regions holding the elemental data for each brick type.

**Data Structures:**
1.  **Root Array:** An array where each element corresponds to a brick. Each element is a struct containing:
    *   `type`: A 4-byte integer identifier for the brick type (Empty, Dense, Voxel List, Octree). Although 2 bits suffice for 4 types, 4 bytes are used for struct alignment [S1, unofficial].
    *   `pointer`: A 4-byte integer (unsigned) pointing to the start of the elemental data. If the volume exceeds $2^{32}$ values, this pointer size is insufficient [S1, unofficial].
2.  **Elemental Data Regions:**
    *   **Dense:** A 1D array storing $m^3 \times d$ bytes per dense brick in row-major order.
    *   **Voxel List:** A 1D array of elements, each containing a 6-byte Morton code (three 2-byte coordinates) and a $d$-byte data value. A separate "offset array" stores a 4-byte pointer to the last element of each voxel list.
    *   **Octree:** A global pointer array for internal nodes (each storing eight 4-byte pointers) and a separate data array for leaf nodes (each storing eight $d$-byte values). Empty nodes are encoded with a special pointer value [S1, unofficial].
    *   **Empty:** No data storage required.

**Algorithm Steps:**

1.  **Sparsity Analysis (Parallel Reductions):**
    For each brick, calculate the memory requirement for all four elemental types. This requires three parallel reductions on the GPU [S1, unofficial]:
    *   **Empty/Voxel List Count:** Perform a pyramidal summation reduction on a binary mask of the brick (1 for non-zero, 0 for zero) to count non-zero voxels $n$. If $n=0$, the brick is Empty [S1, unofficial].
    *   **Octree Internal Nodes:** Count internal octree nodes $n_{internal}$ using a specific reduction on the binary mask.
    *   **Octree Leaf Nodes:** Count leaf nodes $n_{leaf}$ (parent nodes of leaves) using a distinct reduction array [S1, unofficial].

2.  **Memory Calculation:**
    Calculate the size in bytes for each type using the following formulas [S1, unofficial]:
    *   **Empty:** $0$
    *   **Dense:** $d \times m^3$
    *   **Octree:** $32 \times n_{internal} + 8 \times d \times n_{leaf}$
    *   **Voxel List:** $(6 + d) \times n$

3.  **Type Selection (Memory Optimal):**
    Select the brick type with the minimal calculated size for each brick [S1, unofficial].

4.  **Memory Allocation:**
    Sum the sizes of all selected brick types to determine the total memory requirement for each elemental region. Allocate GPU memory accordingly [S1, unofficial].

5.  **Initialization and Upload:**
    Sequentially initialize each brick. For non-empty bricks, populate the elemental data structures (dense array, voxel list with Morton codes, or octree nodes) and store the resulting pointers in the root array [S1, unofficial].

**Lookup Procedure:**
1.  Translate the 3D coordinate to a brick index.
2.  Retrieve `(type, pointer)` from the root array.
3.  If `type` is Empty, return 0.
4.  If `type` is Dense, perform $O(1)$ array access using the pointer.
5.  If `type` is Voxel List, use the offset array to find the range, then perform binary search on Morton codes ($O(\log n)$) [S1, unofficial].
6.  If `type` is Octree, traverse from the root pointer to a leaf ($O(\log m)$ depth), checking for special empty pointers [S1, unofficial].

**Performance Optimization Strategies (Optional):**
If a balance between memory and performance is desired, the memory-optimal selection can be adjusted using the following strategies [S1, unofficial]:
*   **max(ms):** Replace the slowest non-dense brick with Dense first.
*   **max(ms - ms_dense):** Replace bricks with the largest performance gain (difference between actual run time and expected dense run time).
*   **min(byte_dense - byte):** Replace bricks with the smallest memory increase first (globally largest `byte_dense - byte`).
*   **min((byte_dense - byte) / (ms - ms_dense)):** Replace the brick with the smallest slope in the (memory, time) space. This theoretically optimizes the trade-off. Note that measuring GPU performance is noisy, and these strategies rely on local measurements rather than global re-evaluation [S1, unofficial].

## Subtopics

### Dense Volume Brick
A brick type storing all data values (including zeros) in a single 1D array using row-major order. It offers $O(1)$ access time and zero bit overhead per data entry in the best case (full volume), but high overhead in sparse cases. It can be stored as a 3D texture for hardware trilinear interpolation [S1, unofficial].

### Empty Brick
A brick type used when a brick contains only zero values. It stores only its type identifier and returns zero on access, providing the highest access performance and lowest overhead among the four types [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Construction:** Requires three parallel reductions per brick for sparsity analysis. The selection step is $O(1)$ per brick after analysis.
*   **Lookup:** $O(1)$ for Dense/Empty, $O(\log n)$ for Voxel List, $O(\log m)$ for Octree [S1, unofficial].
*   **Memory:** Significantly lower than dense storage for sparse data. Overhead ranges between 1.8 and 27.7 bits per non-zero voxel for tested datasets. The total size never exceeded three times the theoretical lower bound (percentage of non-zero values) [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Adapts to local sparsity; outperforms dense bricking and traditional empty/non-empty bricking for most sparse datasets; allows hardware-friendly dense regions [S1, unofficial].
*   **Limitations:** Introduces branching overhead on the GPU due to type checks; pointer size is fixed at 4 bytes (limiting addressable space to $2^{32}$ values); assumes no large homogeneous non-empty regions (no compression for constant values) [S1, unofficial].
*   **Brick Size:** Optimal brick sizes are typically $8 \times 8 \times 8$ or $16 \times 16 \times 16$. Smaller bricks increase overhead; larger bricks reduce adaptability [S1, unofficial].

## Background
The method was introduced in a thesis by [S1, unofficial] as a solution to the memory inefficiency of traditional bricking, which wastes space on sparse non-empty regions. It combines dense grids, octrees, and voxel lists within a bricking framework to optimize memory consumption and performance for sparse and dense volume data [S1, unofficial].

## Sources
[S1] Eine adaptive, hybride Datenstruktur für Volumen Datensätze auf der GPU, https://doi.org/10.34726/hss.2016.26562, fetched 2026-10-26 (unofficial)
