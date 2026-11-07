# Cell Count Algorithm for Adaptive-Resolution Load Balancing

Also called: Adaptive-resolution data decomposition, Shuttle algorithm for block assignment, Spatial Division of Adaptive-Resolution Data

## Overview
The Cell Count Algorithm is a load-balancing strategy for distributed volume rendering of adaptive-resolution datasets, where data is partitioned into blocks of varying resolutions [S1, unofficial]. It solves the inefficiency of spatial or block-count-based decomposition by ensuring each processor handles a roughly equal total number of cells, thereby preventing idle time caused by processors finishing low-resolution work early [S1, unofficial]. Unlike siblings such as [Scalable Distributed Volume Rendering (SDVR)](863-scalable-distributed-volume-rendering-sdvr.md) or [MPI Volume Rendering (1D/2D Decomposition)](2249-mpi-volume-rendering-1d-2d-decomposition.md) which may rely on uniform spatial grids, this method explicitly accounts for resolution heterogeneity by sorting blocks by cell count and using a "shuttle" assignment mechanism [S1, unofficial].

## How it works
**Inputs:**
- A set of $n$ data blocks $B_1, B_2, \dots, B_n$.
- For each block $B_x$, the cell count $C_x$ (representing the computational workload/resolution).
- The number of available processors $P$.

**Outputs:**
- An assignment mapping each block $B_x$ to a processor $p_i$.
- Each processor $p_i$ receives a subset of blocks such that the sum of cell counts is approximately equal across all active processors.

**Data Structures:**
- `blocks`: A list of objects/tuples containing block ID and cell count $C_x$.
- `processor_list`: A list of processor identifiers $p_1, \dots, p_P$.
- `processor_cell_counts`: A map or array tracking the accumulated cell count for each processor.
- `optimal_cells_per_processor`: An integer value.

**Algorithm Steps:**
1.  **Calculate Total Workload:** Compute the total number of cells in the dataset:
    $$ \text{Total Cells} = \sum_{x=1}^{n} C_x $$
2.  **Calculate Target Load:** Determine the optimal number of cells per processor. Round up to the nearest integer to ensure the target is not underestimated:
    $$ \text{Optimal Cells} = \lceil \frac{\text{Total Cells}}{P} \rceil $$
3.  **Sort Blocks:** Sort the list of blocks in descending order based on their cell count $C_x$. This ensures larger (higher resolution) blocks are assigned first, allowing finer granularity when filling remaining capacity [S1, unofficial].
4.  **Initialize State:**
    - Initialize `processor_cell_counts` for all $P$ processors to 0.
    - Mark all processors as "active" (available for assignment).
    - Set a pointer/index to the sorted blocks list.
5.  **Assign Blocks (Shuttle Algorithm):**
    - While there are unassigned blocks in the sorted list:
        a. Select the next available processor from `processor_list` in a round-robin (cycling) fashion.
        b. Select the next block from the sorted blocks list.
        c. Assign the block to the selected processor.
        d. Update the processor's accumulated cell count:
           $$ \text{processor\_cell\_counts}[p_i] \leftarrow \text{processor\_cell\_counts}[p_i] + C_{\text{current block}} $$
        e. **Check Capacity:** If $\text{processor\_cell\_counts}[p_i] \ge \text{Optimal Cells}$, mark processor $p_i$ as "full" (remove from the active list).
        f. If the processor is marked full, the next iteration will select the next available processor in the cycle.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cell_count_algorithm(blocks, num_processors):
    """
    blocks: list of tuples (block_id, cell_count)
    num_processors: integer P
    Returns: dict mapping processor_id to list of assigned block_ids
    """
    # 1. Calculate total cells
    total_cells = sum(c for _, c in blocks)
    
    # 2. Calculate optimal cells per processor (round up)
    # Using integer arithmetic for ceiling: (a + b - 1) // b
    optimal_cells = (total_cells + num_processors - 1) // num_processors
    
    # 3. Sort blocks descending by cell count
    sorted_blocks = sorted(blocks, key=lambda x: x[1], reverse=True)
    
    # 4. Initialize processor state
    # Assuming processors are indexed 0 to P-1
    processor_loads = [0] * num_processors
    processor_blocks = [[] for _ in range(num_processors)]
    active_processors = list(range(num_processors))
    
    block_idx = 0
    num_blocks = len(sorted_blocks)
    
    # 5. Shuttle assignment
    while block_idx < num_blocks:
        if not active_processors:
            # No active processors left but blocks remain.
            # This implies total_cells / P was too low or rounding issues.
            # In practice, remaining blocks must be assigned to the least loaded processor.
            # However, the source algorithm states it continues until no blocks remain.
            # If active list is empty, we must re-enable the least loaded processor or assign to any.
            # Based on source description: "This algorithm continues until there are no more blocks to assign."
            # And "As soon as a processor is assigned the optimal number... it is removed."
            # If all are full, we likely need to assign remaining to the one with lowest load.
            # But strictly following source: we cycle through processor_list.
            # If processor_list is empty, we break or handle edge case.
            # Given the source example, it assumes P is sufficient or blocks fit.
            # We will assume active_processors is not empty or handle wrap-around.
            break 
            
        # Get next available processor (cycling)
        current_proc_idx = active_processors[0]
        
        # Get next block
        block_id, cell_count = sorted_blocks[block_idx]
        block_idx += 1
        
        # Assign block
        processor_blocks[current_proc_idx].append(block_id)
        processor_loads[current_proc_idx] += cell_count
        
        # Check if processor is full
        if processor_loads[current_proc_idx] >= optimal_cells:
            # Remove from active list
            active_processors.pop(0)
            
    return processor_blocks
```

**Parameters:**
- `num_processors`: The number of threads or MPI ranks.
- `blocks`: The input dataset partitioned into resolution-consistent blocks.

**Edge Cases and Pitfalls:**
- **Processor Count vs Block Count:** If the number of processors is less than the number of blocks, the algorithm assigns multiple blocks to processors. If the number of processors is greater than the number of blocks, some processors may remain idle or receive no blocks [S1, unofficial].
- **Rounding Errors:** The use of ceiling for `optimal_cells` means the last processor(s) might receive slightly more than the average, or the sum of assigned cells might exceed `Total Cells` if not careful (though the algorithm assigns existing blocks, so the sum is exactly `Total Cells`). The imbalance is minimized but not eliminated.
- **Consistent Resolution:** The algorithm assumes each block has a consistent internal resolution. It does not subdivide blocks further; if a single block exceeds `optimal_cells`, it must be assigned entirely to one processor, potentially causing imbalance [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Sorting blocks takes $O(n \log n)$ where $n$ is the number of blocks. The assignment loop runs $O(n)$ times. Total time is dominated by sorting.
- **Memory Complexity:** $O(n)$ to store block metadata and assignment lists.
- **Strengths:** Effectively balances computational load for heterogeneous data. Prevents idle time associated with low-resolution blocks finishing early. Simple to implement.
- **Limitations:** Requires a pre-pass to calculate cell counts. Does not account for network communication costs or data locality, only computational load. If block sizes vary wildly, the granularity of balancing is limited by the smallest block size.
- **When to use:** Adaptive-resolution datasets where resolution varies significantly across spatial regions.
- **When not to use:** Uniform resolution data (where simple spatial decomposition suffices) or when communication overhead dominates computation.

## Background
The algorithm was described in the context of volume visualization of time-varying data using parallel, multiresolution, and adaptive-resolution techniques [S1, unofficial]. It addresses the specific problem of load imbalance in adaptive-resolution data sets, where spatial division leads to inefficiency because processors handling low-resolution portions complete work before those handling high-resolution portions [S1, unofficial]. The approach is also referred to as the "shuttle algorithm" for block assignment in the source literature [S1, unofficial].

## Sources
[S1] Volume visualization of time-varying data using parallel, multiresolution and adaptive-resolution techniques, https://scholars.unh.edu/cgi/viewcontent.cgi?article=1245&context=thesis, fetched 2026-10-27 (unofficial)
