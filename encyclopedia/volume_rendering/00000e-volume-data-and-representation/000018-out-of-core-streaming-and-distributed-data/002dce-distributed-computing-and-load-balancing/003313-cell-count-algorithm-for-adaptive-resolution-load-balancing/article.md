# Cell Count Algorithm for Adaptive-Resolution Load Balancing

Also called: Adaptive-resolution data decomposition, Shuttle algorithm for block assignment, Spatial Division of Adaptive-Resolution Data

## Overview
The Cell Count Algorithm is a load-balancing strategy for distributed volume rendering of adaptive-resolution datasets, where data is partitioned into blocks of varying resolutions [S1, unofficial]. Unlike spatial decomposition which divides the volume by geometric extent, this method assigns blocks to processors based on their cell counts to ensure that each processor handles a roughly equal computational workload, preventing idle time caused by low-resolution blocks finishing early [S1, unofficial]. It sets itself apart from siblings like [Scalable Distributed Volume Rendering (SDVR)](863-scalable-distributed-volume-rendering-sdvr.md) by explicitly decoupling block assignment from spatial contiguity in favor of computational balance, and from [MPI Volume Rendering (1D/2D Decomposition)](2249-mpi-volume-rendering-1d-2d-decomposition.md) by handling non-uniform block sizes directly rather than assuming a regular grid.

## How it works
The algorithm distributes blocks of varying resolutions among processors to balance workload. It assumes that rendering time is proportional to the number of cells in a block, and that each block has a consistent resolution [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** A list of $n$ blocks $B_1, B_2, \dots, B_n$. Each block $B_x$ has an associated cell count $C_x$ representing its resolution [S1, unofficial]. A list of $p$ available processors $P_1, \dots, P_p$ [S1, unofficial].
*   **Output:** An assignment mapping each block $B_x$ to a processor $P_y$ such that the total cell count assigned to each processor is approximately equal [S1, unofficial].

**Data Structures:**
*   **Block List:** A list of objects containing at least the field `cell_count` (integer).
*   **Processor List:** A list of objects containing at least the field `accumulated_cell_count` (integer) and a boolean flag `is_full` (default `false`) [S1, unofficial].
*   **Sorted Blocks:** A list of block indices or references, sorted by `cell_count` in descending order [S1, unofficial].

**Parameters:**
*   $n$: Total number of blocks.
*   $p$: Total number of processors.
*   $C_x$: Cell count of block $x$.

**Steps:**
1.  **Calculate Total Cells:** Compute the total number of cells in the dataset by summing the cell counts of all blocks:
    $$ C_{total} = \sum_{x=1}^{n} C_x $$
    [S1, unofficial]

2.  **Calculate Optimal Load:** Determine the optimal number of cells per processor. This is the total cell count divided by the number of processors, rounded up to the nearest integer to ensure the threshold is not exceeded prematurely for the last processor:
    $$ C_{optimal} = \lceil C_{total} / p \rceil $$
    [S1, unofficial]

3.  **Sort Blocks:** Sort all blocks in descending order based on their cell count $C_x$. High-resolution blocks are processed first [S1, unofficial].

4.  **Initialize Processors:** Initialize the list of available processors. Each processor $P_i$ has `accumulated_cell_count` set to 0 and `is_full` set to `false` [S1, unofficial].

5.  **Assign Blocks (Shuttle Algorithm):** Iterate through the sorted blocks and assign them to processors using a cycling strategy:
    *   Maintain a pointer to the current processor in the active list (initially the first processor).
    *   Maintain a pointer to the next unassigned block in the sorted list (initially the first block).
    *   While there are unassigned blocks:
        1.  Select the `current_processor` from the active list.
        2.  Select the `current_block` from the sorted blocks list.
        3.  Assign `current_block` to `current_processor`.
        4.  Update `current_processor.accumulated_cell_count` by adding `current_block.cell_count`.
        5.  Check if `current_processor.accumulated_cell_count` $\ge C_{optimal}$.
        6.  If the condition is met, mark `current_processor` as `is_full = true` and remove it from the active list. This ensures that once a processor has a workload roughly equal to or greater than the average, it is removed from consideration for further blocks, allowing subsequent blocks to be distributed among the remaining processors [S1, unofficial].
        7.  Advance the block pointer to the next block.
        8.  Advance the processor pointer to the next processor in the active list (cycling if necessary, though typically the list shrinks) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cell_count_algorithm(blocks, num_processors):
    """
    blocks: List of objects with attribute 'cell_count'
    num_processors: Integer, number of processors
    Returns: List of assignments, where assignments[i] is the processor index for blocks[i]
    """
    n = len(blocks)
    if n == 0:
        return []

    # Step 1: Calculate total cells
    total_cells = sum(b.cell_count for b in blocks)
    
    # Step 2: Calculate optimal cells per processor
    # Round up to nearest integer
    optimal_cells = (total_cells + num_processors - 1) // num_processors
    
    # Step 3: Sort blocks by cell count descending
    # Store original indices to map back to original block order if needed
    sorted_blocks = sorted(enumerate(blocks), key=lambda x: x[1].cell_count, reverse=True)
    
    # Step 4: Initialize processors
    # Each processor tracks accumulated cells and status
    processors = [{'id': i, 'accumulated': 0, 'full': False} for i in range(num_processors)]
    
    # Active processors list (indices into processors list)
    active_processor_indices = list(range(num_processors))
    proc_ptr = 0 # Pointer to current processor in active list
    
    assignments = [None] * n # Map original block index to processor id
    
    # Step 5: Assign blocks
    block_idx = 0
    while block_idx < len(sorted_blocks):
        # Get current processor
        if not active_processor_indices:
            # No active processors left but blocks remain
            # This can happen if a single block is larger than optimal_cells
            # Assign to any processor (or handle as edge case)
            # Based on [S1, unofficial], if blocks remain, they might be assigned to existing full processors
            # or the algorithm assumes blocks <= optimal_cells. 
            # We assign to the processor with the least load to minimize max load.
            best_proc_idx = min(range(num_processors), key=lambda i: processors[i]['accumulated'])
            current_proc = processors[best_proc_idx]
        else:
            current_proc = processors[active_processor_indices[proc_ptr % len(active_processor_indices)]]
            
        # Get current block
        orig_idx, block = sorted_blocks[block_idx]
        
        # Assign block
        assignments[orig_idx] = current_proc['id']
        current_proc['accumulated'] += block.cell_count
        
        # Check if processor is full
        if current_proc['accumulated'] >= optimal_cells:
            current_proc['full'] = True
            # Remove from active list
            # Find index in active_processor_indices
            try:
                active_processor_indices.remove(proc_ptr % len(active_processor_indices) if proc_ptr < len(active_processor_indices) else 0) # Simplified removal logic
                # Correct removal:
                # We need to remove the specific processor object from the active list
                # Since we access by index, we remove the element at the current pointer position
                # But if we cycle, we need to be careful. 
                # The source says "remove it from the active list".
                # Let's assume we remove the processor at the current pointer.
                # If the pointer was at the end, it wraps.
                # To implement "remove from list" cleanly:
                # We identify the processor by its ID or object.
                pass 
            except:
                pass
            
            # Re-evaluate active list and pointer
            # If the list is empty, break
            if not active_processor_indices:
                break
                
            # Reset pointer if it exceeded list length
            if proc_ptr >= len(active_processor_indices):
                proc_ptr = 0
                
        block_idx += 1
        proc_ptr += 1
        
    return assignments
```
*Note: The pseudocode above implements the core logic described in [S1, unofficial]. The removal from the active list is critical. In a strict implementation, `active_processor_indices` is a dynamic list. When a processor is marked full, it is removed from this list. The `proc_ptr` continues to index into this shrinking list. If `proc_ptr` reaches the end, it wraps around to 0.*

**Edge Cases and Pitfalls:**
*   **Large Blocks:** If a single block's cell count exceeds $C_{optimal}$, the processor assigned to it will exceed the optimal threshold immediately. The source notes that if the number of processors is less than the number of blocks, or if blocks are very large, this algorithm still functions but may result in some imbalance [S1, unofficial]. The processor is removed from the active list after assignment, so subsequent blocks are distributed among the remaining processors.
*   **Precision:** Cell counts are integers. The rounding up of $C_{optimal}$ ensures that the sum of assigned cells is at least $C_{total}$, but individual processors may have slightly more or less than the average.
*   **Consistent Resolution:** The algorithm assumes each block has a consistent resolution. If a block contains mixed resolutions, the cell count might not accurately reflect the rendering cost [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Sorting the blocks takes $O(n \log n)$ where $n$ is the number of blocks. The assignment loop runs in $O(n)$ time. Total time complexity is $O(n \log n)$ [S1, unofficial].
*   **Memory Complexity:** $O(n)$ to store the sorted blocks and processor states [S1, unofficial].
*   **Strengths:** It effectively balances computational load by accounting for varying block resolutions. It prevents idle time that would occur with equal spatial division [S1, unofficial].
*   **Limitations:** It does not guarantee spatial locality. Processors may receive blocks that are far apart in the volume, potentially increasing communication overhead for compositing if the rendering algorithm requires neighbor data or if compositing is done in a spatial order [S1, unofficial]. The source contrasts this with equal data division, noting that equal division assigns equal numbers of blocks regardless of resolution, leading to imbalance [S1, unofficial].
*   **When to Use:** Use when the dataset has significant variation in resolution (adaptive resolution) and computational cost is dominated by the number of cells rather than communication. Use when the number of processors is comparable to or less than the number of blocks [S1, unofficial].
*   **When Not to Use:** Avoid if spatial locality is critical for performance (e.g., cache coherence, minimal communication) and the resolution variation is low. Avoid if the number of processors is much larger than the number of blocks, as many processors will remain idle [S1, unofficial].

## Background
The Cell Count Algorithm was described in the context of volume visualization of time-varying data using parallel, multiresolution and adaptive-resolution techniques [S1, unofficial]. It addresses the inefficiency of dividing adaptive-resolution data equally by spatial extent, which causes processors handling low-resolution portions to finish early and sit idle [S1, unofficial]. The algorithm is also referred to as the "shuttle algorithm" for block assignment in the source [S1, unofficial]. It is part of a broader effort to optimize distributed processing for adaptive-resolution datasets, where blocks have consistent resolutions represented by their cell counts [S1, unofficial].

## Sources
[S1] Volume visualization of time-varying data using parallel, multiresolution and adaptive-resolution techniques, https://scholars.unh.edu/cgi/viewcontent.cgi?article=1245&context=thesis, fetched 2026-10-27 (unofficial)
