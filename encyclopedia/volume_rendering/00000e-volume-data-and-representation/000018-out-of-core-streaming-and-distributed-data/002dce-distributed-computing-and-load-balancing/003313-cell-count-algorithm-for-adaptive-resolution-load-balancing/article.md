# Cell Count Algorithm for Adaptive-Resolution Load Balancing

Also called: Adaptive-resolution data decomposition, Shuttle algorithm for block assignment, Spatial Division of Adaptive-Resolution Data

## Overview
The Cell Count Algorithm is a load-balancing strategy for distributed volume rendering of adaptive-resolution datasets, where data is partitioned into blocks of varying resolutions [S1, unofficial]. Unlike uniform decomposition, which assigns equal spatial extents or block counts and causes idle time for processors handling low-resolution regions, this method balances computational workload by ensuring each processor handles approximately the same total number of voxels (cells) [S1, unofficial]. It sets itself apart from siblings like [Scalable Distributed Volume Rendering (SDVR)](863-scalable-distributed-volume-rendering-sdvr.md) by explicitly decoupling block assignment from spatial contiguity in favor of cell-count equality, and from [Dynamic load balancing using kd-trees](9365-dynamic-load-balancing-using-kd-trees.md) by using a static, greedy sorting approach rather than recursive spatial partitioning.

## How it works
**Inputs:**
*   A set of $n$ data blocks $B = \{B_1, B_2, \dots, B_n\}$.
*   For each block $B_i$, the cell count $C_i$ (representing its resolution/computational cost).
*   The number of processors $P$.

**Outputs:**
*   An assignment map mapping each processor $p_j$ to a subset of blocks $B_{assigned}$.

**Data Structures:**
*   `Block`: A structure containing at least `id` and `cell_count`.
*   `Processor`: A structure containing `id`, `current_cell_count` (accumulator), and `is_full` (boolean flag).
*   `sorted_blocks`: A list of `Block` objects sorted in descending order of `cell_count`.
*   `processor_list`: A list of `Processor` objects, initially all with `current_cell_count = 0` and `is_full = false`.

**Steps:**
1.  **Calculate Total Cells:** Compute the sum of cell counts for all blocks:
    $$ C_{total} = \sum_{i=1}^{n} C_i $$
2.  **Calculate Optimal Load:** Determine the target number of cells per processor:
    $$ C_{optimal} = \lceil C_{total} / P \rceil $$
    [G] Rounding up ensures that if $C_{total}$ is not perfectly divisible by $P$, the last processor absorbs the remainder without under-assigning relative to the integer ceiling.
3.  **Sort Blocks:** Sort the list of blocks $B$ in descending order of $C_i$. This prioritizes assigning the most expensive (highest resolution) blocks first to prevent them from being fragmented across many processors or left for the end where load balancing is harder.
4.  **Initialize Processors:** Create a list of $P$ processors.
5.  **Assign Blocks (Shuttle Algorithm):**
    *   Iterate through the `sorted_blocks` list.
    *   For each block, select the next available processor from `processor_list`. The selection cycles through processors $p_1, p_2, \dots, p_P, p_1, \dots$ [S1, unofficial].
    *   Assign the block to the selected processor.
    *   Update the processor's `current_cell_count` by adding the block's $C_i$.
    *   **Check Capacity:** If `current_cell_count` $\ge C_{optimal}$, mark the processor as `is_full = true` and remove it from the active rotation of `processor_list`.
    *   Repeat until all blocks are assigned.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cell_count_algorithm(blocks, num_processors):
    # blocks: list of dicts/objects with 'id' and 'cell_count'
    # num_processors: integer P
    
    # 1. Calculate total cells
    total_cells = sum(b['cell_count'] for b in blocks)
    
    # 2. Calculate optimal cells per processor (round up)
    optimal_cells = (total_cells + num_processors - 1) // num_processors
    
    # 3. Sort blocks descending by cell count
    sorted_blocks = sorted(blocks, key=lambda b: b['cell_count'], reverse=True)
    
    # 4. Initialize processors
    processors = []
    for i in range(num_processors):
        processors.append({
            'id': i,
            'assigned_blocks': [],
            'current_cell_count': 0,
            'is_full': False
        })
    
    # Active list of processors for cycling
    active_processors = list(processors)
    
    # 5. Assign blocks
    for block in sorted_blocks:
        if not active_processors:
            # Should not happen if num_processors >= 1 and blocks exist,
            # but if processors run out, assign to the one with least load [G]
            # or ignore if P > n. Based on source, we assume P <= n or blocks are exhausted.
            break
            
        # Get next available processor (cycling)
        current_processor = active_processors.pop(0)
        
        # Assign block
        current_processor['assigned_blocks'].append(block)
        current_processor['current_cell_count'] += block['cell_count']
        
        # Check if processor is full
        if current_processor['current_cell_count'] >= optimal_cells:
            current_processor['is_full'] = True
            # Do not add back to active list
        else:
            # Add back to end of list for next turn
            active_processors.append(current_processor)
            
    return processors
```

**Edge Cases and Pitfalls:**
*   **Processor Count vs. Block Count:** If the number of processors $P$ exceeds the number of blocks $n$, some processors will remain idle. The source notes that "extra processors can be ignored" [S1, unofficial].
*   **Block Subdivision:** If a single block's cell count exceeds $C_{optimal}$, the algorithm as described does not subdivide it. The source mentions "blocks can be subdivided" as a potential resolution, but the core algorithm assumes atomic block assignment [S1, unofficial].
*   **Consistency Constraint:** The algorithm prioritizes assigning whole blocks to maintain "consistent resolution" within a processor's workload, which simplifies rendering logic but may lead to slight over-assignment (load > $C_{optimal}$) for the last blocks assigned to a processor.

## Subtopics

### Adaptive-Resolution Data Decomposition
The general category of splitting data based on resolution rather than spatial volume. Chosen when data has significant variance in detail density. Link: [Adaptive-resolution data decomposition](863-scalable-distributed-volume-rendering-sdvr.md) (contextual sibling).

### Shuttle Algorithm for Block Assignment
The specific greedy assignment mechanism used here. Chosen for its simplicity and O(N log N) sorting cost. Link: [Shuttle algorithm](9702-load-balanced-network-distribution-lbnd.md) (conceptual sibling).

## Operating characteristics
*   **Time Complexity:** Dominated by sorting the blocks, $O(n \log n)$, where $n$ is the number of blocks. The assignment loop is $O(n)$.
*   **Memory Complexity:** $O(n + P)$ to store block metadata and processor states.
*   **Performance:** Significantly reduces idle time compared to uniform spatial decomposition in adaptive datasets. In the source example with 4 processors and blocks [16, 16, 4, 4, 4, 4, 1, 1], the algorithm assigns loads of 16, 16, 9, and 9 cells, whereas equal block division might assign 17, 20, etc., leading to imbalance [S1, unofficial].
*   **Strengths:** Simple to implement; effective at balancing load for static adaptive-resolution data; prevents low-resolution processors from finishing early.
*   **Limitations:** Does not guarantee perfect balance (remainder cells are unevenly distributed); requires global knowledge of all block sizes before assignment (static); does not account for view-dependent visibility (e.g., if a high-res block is occluded, its cost is wasted).
*   **When to Use:** Static adaptive-resolution datasets where block boundaries are fixed and view-independent load balancing is sufficient.
*   **When Not to Use:** Dynamic datasets where block resolutions change frequently at runtime, or when view-dependent culling is the primary optimization.

## Background
The algorithm was described in a thesis by researchers at the University of New Hampshire, focusing on volume visualization of time-varying data using parallel, multiresolution, and adaptive-resolution techniques [S1, unofficial]. It addresses the inefficiency of dividing adaptive-resolution data equally by spatial extent, which causes processors handling low-resolution portions to complete work early and sit idle [S1, unofficial]. The method is part of a broader effort to optimize distributed processing for medical and scientific imaging where resolution varies across the volume.

## Sources
[S1] Volume visualization of time-varying data using parallel, multiresolution and adaptive-resolution techniques, https://scholars.unh.edu/cgi/viewcontent.cgi?article=1245&context=thesis, fetched 2026-10-27 (unofficial)
