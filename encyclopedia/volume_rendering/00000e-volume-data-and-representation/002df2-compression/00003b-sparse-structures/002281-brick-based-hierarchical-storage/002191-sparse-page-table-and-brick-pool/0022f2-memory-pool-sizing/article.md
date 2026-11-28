# Memory Pool Sizing
## Overview
Memory pool sizing defines the capacity and allocation limits for the node pool and brick pool in sparse volume representations, bounding memory usage while enabling the referencing of large logical volumes. It sets apart from dense grid approaches by decoupling logical resolution from resident memory, allowing full source resolution (up to $2048^3$ cells per axis) while skipping empty space [S1, unofficial].
## How it works
### Inputs and outputs
- **Inputs**: Target logical volume dimensions, brick resolution, and available memory budget.
- **Outputs**: Configured pool sizes (number of entries/bricks) and memory allocation parameters.

### Data structures
The method relies on two primary pools defined in the parent structure [Sparse Page Table and Brick Pool](2550-sparse-page-table-and-brick-pool.md):
1.  **Node Pool**: A sparse page table mapping logical regions to resident bricks.
    -   **Layout**: Array of indices or pointers.
    -   **Size**: Determined by the logical grid resolution and brick size.
2.  **Brick Pool**: A pool of apron-padded bricks containing actual volume data.
    -   **Layout**: Array of brick structures.
    -   **Size**: Determined by available memory and brick data size.

### Steps
1.  **Determine Logical Resolution**: Define the maximum logical volume dimensions $L_x, L_y, L_z$ [S1, unofficial].
2.  **Select Brick Resolution**: Choose brick dimensions $B_x, B_y, B_z$ (e.g., $16^3$) [S1, unofficial].
3.  **Calculate Node Pool Size**:
    -   Compute the number of logical bricks required: $N_{nodes} = \lceil L_x/B_x \rceil \times \lceil L_y/B_y \rceil \times \lceil L_z/B_z \rceil$ [S1, unofficial].
    -   Allocate the node pool to hold $N_{nodes}$ entries [S1, unofficial].
4.  **Calculate Brick Pool Size**:
    -   Determine the memory budget for bricks $M_{budget}$ [S1, unofficial].
    -   Compute the size of a single brick $S_{brick}$ (including apron padding) [S1, unofficial].
    -   Calculate maximum resident bricks: $N_{bricks} = \lfloor M_{budget} / S_{brick} \rfloor$ [S1, unofficial].
    -   Allocate the brick pool to hold $N_{bricks}$ bricks [S1, unofficial].
5.  **Initialize Pools**:
    -   Initialize node pool entries to null/invalid [G].
    -   Initialize brick pool entries as free [G].

### Formulas
-   **Logical Indices Referenced**: $I_{total} = N_{nodes}$ [S1, unofficial].
-   **Brick Capacity**: $C_{bricks} = N_{bricks}$ [S1, unofficial].

### Parameters
-   **Node Pool Size**: Typically small, e.g., 4 MB, corresponding to $64^3$ entries [S1, unofficial].
-   **Brick Pool Size**: e.g., 430 MB, giving room for $42^3$ bricks [S1, unofficial].
-   **Brick Resolution**: e.g., $16^3$ [S1, unofficial].
-   **Logical Resolution**: e.g., $1024^3$ indices referenced [S1, unofficial].

### Edge cases and pitfalls
-   **Memory Exhaustion**: If the logical volume is too large for the node pool, mapping fails.
-   **Thrashing**: If the brick pool is too small for the active view, frequent swapping occurs.
-   **Precision**: Ensure brick size calculations account for apron padding correctly to avoid boundary errors during rendering [S1, unofficial].

## Subtopics

### Memory Pooling for Chunk Arrays
Recycles typed arrays from the main thread to avoid garbage collection per edit, storing them in `chunkPool` or `windowPool` based on length [S2, unofficial].
-   **Step**:
    1.  Check if `msg.recycle` exists [S2, unofficial].
    2.  For each array `a` in `recycle`, determine pool: `a.length === N3 ? chunkPool : a.length === P3 ? windowPool : null` [S2, unofficial].
    3.  If pool exists and `pool.length < 256`, push `a` into pool [S2, unofficial].
    4.  Reuse pooled arrays for future chunk/window allocations [S2, unofficial].
-   **Parameters**:
    -   `P3`: $(CHUNK + 2 \times PAD)^3$, size for padded mesher windows [S2, unofficial].
    -   `pool capacity`: Maximum number of arrays retained per pool. Default: 256 [S2, unofficial].

## Operating characteristics
-   **Time Complexity**: Allocation is $O(1)$ for pool lookups [G].
-   **Memory Complexity**: Bounded by pool sizes (e.g., 4 MB + 430 MB) [S1, unofficial].
-   **Performance**: Enables interactive rendering of large volumes (e.g., $2048^3$) on limited hardware (e.g., 512 MB GPU) [S1, unofficial].
-   **Strengths**: Memory efficiency, scalability to large logical volumes.
-   **Limitations**: Requires careful tuning of pool sizes to avoid thrashing or overflow.

## Background
Originated in Ray-Guided Streaming for Efficient and Detailed Voxel Rendering (2009) [S1, unofficial]. Used in interactive volume rendering systems to manage large voxel datasets on GPUs with limited memory [S1, unofficial].

## Sources
[S1] [PDF] Ray-Guided Streaming for Efficient and Detailed Voxel Rendering, https://www-sop.inria.fr/reves/Basilic/2009/CNLE09/CNLE09.pdf, fetched 2026-10-26 (unofficial)
[S2] sh1ftmaker/voxel-wars, https://github.com/sh1ftmaker/voxel-wars, fetched 2026-10-28 (unofficial)
