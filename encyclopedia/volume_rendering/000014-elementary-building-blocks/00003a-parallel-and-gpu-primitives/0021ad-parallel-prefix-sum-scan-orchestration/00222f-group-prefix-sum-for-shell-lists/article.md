# Group Prefix Sum for Shell Lists


## Overview
Group Prefix Sum for Shell Lists is a parallel compaction method for shell lists in GPU-based isosurface extraction, replacing sequential thread-0 processing with a parallel prefix sum approach. It solves the bottleneck of sequentially compacting shell lists by enabling all threads to participate in the compaction process simultaneously. Unlike standard exclusive prefix sum implementations, this method operates on grouped shell data structures, coordinating with the octree to determine block offsets from the first active octree nodes (AOs) in each shell.

## How it works
The method processes shell lists in parallel groups, where each group handles a block of 128 functions. Shell data is packed into chunks of at most 32 AOs, with rows of coefficient matrices (C) staged in groupshared memory for simultaneous reading by all threads in the group.

**Inputs and Outputs:**
- Input: Shell lists containing AO offsets and coefficient data for up to 128 functions per block
- Output: Compacted shell lists with updated AO offsets
- Auxiliary: Octree block offsets derived from the first AO in each shell

**Data Structures:**
- Shell list entries: Contain AO offsets and coefficient matrix rows
- Groupshared memory: Stores up to 32 AOs per chunk, with 16 field records uploaded per thread
- Octree offsets: Derived from the first AO of each shell in a block

**Steps:**
1. Load shell list data into groupshared memory, uploading all 16 field records per thread
2. Clamp loads from groupshared indices to handle inactive lanes (particularly for llvmpipe compatibility)
3. Compute parallel prefix sum (iso_group_prefix) across the shell list entries
4. Extract AO offsets from the first AOs of each shell to determine block offsets for the octree
5. Update shell list entries with compacted positions based on the prefix sum results

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def group_prefix_sum_shell_lists(shell_data, group_size=128):
    """
    Parallel compaction of shell lists using group prefix sum.
    
    Args:
        shell_data: List of shell entries with AO offsets and coefficients
        group_size: Number of functions per block (default 128)
    
    Returns:
        Compacted shell data with updated offsets
    """
    # Stage data in groupshared memory
    groupshared_data = load_to_groupshared(shell_data, records_per_thread=16)
    
    # Clamp indices for inactive lanes
    clamped_indices = clamp(groupshared_data.indices, min=0, max=len(shell_data)-1)
    
    # Compute parallel prefix sum
    prefix_sums = exclusive_scan(clamped_indices)
    
    # Extract block offsets from first AO of each shell
    block_offsets = extract_first_ao_offsets(shell_data)
    
    # Update shell entries with compacted positions
    compacted_data = update_shell_offsets(shell_data, prefix_sums, block_offsets)
    
    return compacted_data
```

**Parameters:**
- Group size: 128 functions per block
- Chunk size: Maximum 32 AOs per chunk
- Field records: 16 per thread

**Edge Cases:**
- Inactive lanes: Loads from groupshared memory must be clamped to prevent out-of-bounds access
- llvmpipe compatibility: Special handling required for inactive lane loads
- Empty shells: Must handle cases where shells contain no valid AOs

## Subtopics
## Operating characteristics
The method enables parallel processing of shell list compaction, significantly improving performance over sequential thread-0 processing. The use of groupshared memory reduces global memory bandwidth requirements by allowing threads to share data locally. However, the method requires careful handling of inactive lanes and clamping operations, which may introduce additional computational overhead. The approach is particularly effective for large shell lists where parallel processing benefits outweigh the coordination costs.

## Background
This method originates from the md_gto_iso_gpu optimization work by Scanberg, implemented in the mdlib repository. The approach represents a shift from sequential to parallel shell list compaction in GPU-based isosurface extraction, improving performance for large-scale volume rendering applications. The implementation includes specific handling for llvmpipe compatibility and uses environment variables (GTO_ISO_RELAX_FIRST, GTO_ISO_CORNERS_V2) to control behavior.

## Sources
[S1] md_gto_iso_gpu: optimized GPU isosurface extraction (non-blocking jobs, flood, one measured path), https://github.com/scanberg/mdlib/pull/15, fetched 2026-10-28 (unofficial)
