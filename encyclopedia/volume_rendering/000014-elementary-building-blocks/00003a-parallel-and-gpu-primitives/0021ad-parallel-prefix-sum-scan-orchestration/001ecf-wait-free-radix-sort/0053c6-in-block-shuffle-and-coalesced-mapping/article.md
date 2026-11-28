# In-block shuffle and coalesced mapping

Also called: Coalesced Cluster-wise Mapping, In-block shuffle radix counting with coalesced cluster-wise mapping, Modified non-coalesced element-wise mapping

## Overview
This method modifies the shuffle steps of radix sorting to achieve coalesced memory access, addressing the performance penalty of random-like scattering patterns in global memory writes [S1, unofficial]. It sets itself apart from standard element-wise mapping by performing local shuffling in shared memory to cluster same-radix elements, followed by a coalesced cluster-wise mapping to global memory, ensuring that the $i$-th thread outputs data to the $i$-th position in a 128-byte aligned region [S1, unofficial].

## How it works
**Inputs:** An array of values (and optionally indices) residing in global memory, partitioned into blocks processed by a thread block.
**Outputs:** The sorted array in global memory.
**Data Structures:**
*   **Shared Memory Buffer:** A temporary buffer sized to hold one block of data.
*   **Cluster Start Positions:** An array storing the global memory offset of the first element for each unique radix combination (cluster) present in the block. The size of this array equals the number of unique radix combinations (e.g., 16 for 4-bit radix, 256 for 8-bit).

**Algorithm Steps:**
The procedure is executed per radix digit pass. Let $N$ be the number of elements in the block, $B$ be the block size (number of threads), and $R$ be the radix size (number of possible digit values).

1.  **Read to Shared Memory:** All threads in the block read their assigned elements from global memory into the shared memory buffer. This ensures fast, low-latency access for the subsequent shuffle operations [S1, unofficial].
2.  **Compute Cluster Start Positions:**
    *   Each thread determines the radix digit of its element.
    *   Using a parallel prefix sum (scan) or atomic operations within the block, compute the starting global memory offset for each radix cluster.
    *   Crucially, only the start position of the *first* element in each cluster is stored. The position of subsequent elements in the same cluster is defined implicitly based on the order of appearance in the input (due to the stability of LSD radix sort) [S1, unofficial].
3.  **Determine Thread-to-Cluster Mapping:**
    *   For each thread, identify which cluster its element belongs to.
    *   Determine the corresponding thread index that will handle the output for the specific position in the global memory output buffer. This step ensures that consecutive threads in the warp write to consecutive memory addresses.
4.  **Coalesced Cluster-wise Mapping:**
    *   Each thread reads its element from the shared memory buffer.
    *   The thread calculates the final global memory destination address. This address is derived from the cluster start position (computed in Step 2) plus the implicit offset of the element within its cluster.
    *   The thread writes its element to the calculated global memory address.
    *   This step is performed in parallel by all threads, once for each radix combination, ensuring that the $i$-th thread outputs data to the $i$-th position in a 128-byte aligned region [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assume:
# - global_in: input array in global memory
# - global_out: output array in global memory
# - shared_mem: shared memory buffer of size BLOCK_SIZE
# - cluster_starts: shared memory array of size RADIX_SIZE, initialized to 0
# - block_id: unique ID for the current block
# - radix_digit: function returning the current radix digit of a value
# - RADIX_SIZE: number of possible values for the current radix digit (e.g., 16 or 256)
# - BLOCK_SIZE: number of threads in the block

def in_block_shuffle_coalesced_mapping(global_in, global_out, shared_mem, cluster_starts, block_id, radix_digit, RADIX_SIZE, BLOCK_SIZE):
    tid = get_thread_id()
    
    # Step 1: Read data in block to shared memory
    shared_mem[tid] = global_in[block_id * BLOCK_SIZE + tid]
    sync_threads()
    
    # Step 2: Compute the position of first elements in a cluster
    # This typically involves a parallel scan or atomic adds to determine
    # the starting global offset for each radix bucket.
    # For simplicity, assume cluster_starts[r] is computed correctly
    # for each radix value r in [0, RADIX_SIZE-1].
    # The value stored is the global memory index where the first element
    # of radix r should be placed.
    
    # Step 3: Each thread outputs corresponding data to global memory output
    # We exploit the stability of LSD radix sort: elements with the same
    # radix digit maintain their relative order.
    
    val = shared_mem[tid]
    r = radix_digit(val)
    
    # Determine the implicit offset of this element within its cluster.
    # Since we only store the start position, we need to know how many
    # elements with the same radix 'r' have appeared before this thread's element.
    # This can be done via a local scan or by counting.
    # Let's assume a helper function count_occurrences_before(tid, r, shared_mem, radix_digit)
    # which returns the number of elements in shared_mem[0...tid-1] with radix digit r.
    
    offset_in_cluster = count_occurrences_before(tid, r, shared_mem, radix_digit)
    
    # Calculate global destination address
    dest_addr = cluster_starts[r] + offset_in_cluster
    
    # Write to global memory
    # This write is coalesced because threads with consecutive tids
    # that have consecutive elements in the same cluster will write
    # to consecutive memory addresses.
    global_out[dest_addr] = val
    
    sync_threads()
```

**Parameters:**
*   **Alignment Requirement:** Global memory output must be 128-byte aligned to satisfy the coalesced writing condition [S1, unofficial].
*   **Radix Size:** Typically 4-bit (16 buckets) or 8-bit (256 buckets) depending on the hardware and data type.

**Edge Cases and Pitfalls:**
*   **Memory Footprint:** By storing only the first position of each cluster, the memory footprint for tracking positions is reduced compared to storing all mapping positions [S1, unofficial].
*   **Stability Assumption:** The method relies on the stability of the LSD radix sort. If the sort is not stable, the implicit offset calculation will be incorrect.
*   **Shared Memory Contention:** High contention on shared memory during the read and cluster start computation steps can reduce performance.

## Operating characteristics
*   **Time Complexity:** The method adds an $O(N)$ shuffling step in shared memory but reduces the global memory write latency by ensuring coalesced accesses.
*   **Memory Complexity:** Requires $O(N)$ shared memory for the block data and $O(R)$ shared memory for cluster start positions, where $R$ is the radix size.
*   **Performance:** Provides a 50% performance improvement on graphics cards supporting CUDA 1.0 and 1.1 (e.g., G80 and GeForce Quadro FX generation) [S1, unofficial]. On newer cards (e.g., GTX 280), it provides a 10% improvement [S1, unofficial].
*   **Strengths:** Significantly reduces the penalty of non-coalesced writing, which can account for up to 36% of sorting time in standard implementations [S1, unofficial].
*   **Limitations:** Requires additional logic to compute cluster start positions and implicit offsets. The benefit diminishes on newer hardware with better memory controllers.

## Background
This technique was proposed to address the inefficiency of random-like scattering patterns in GPU radix sorting, which lead to non-coalesced writing [S1, unofficial]. It is based on the observation that LSD radix sorting is stable, preserving the order of numbers with the same radix combination bits [S1, unofficial]. The method was introduced in the context of fast 4-way parallel radix sorting on GPUs [S1, unofficial].

## Sources
[S1] Fast 4-way parallel radix sorting on GPUs, https://www.sci.utah.edu/~csilva/papers/cgf.pdf, fetched 2026-10-29 (unofficial)
