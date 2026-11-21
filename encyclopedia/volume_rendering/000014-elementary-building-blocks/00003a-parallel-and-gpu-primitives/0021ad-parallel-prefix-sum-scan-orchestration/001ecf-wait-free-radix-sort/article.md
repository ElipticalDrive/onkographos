# Wait-Free Radix Sort

Also called: Hierarchical Blelloch Scan Sort

## Overview
A parallel sorting algorithm designed for GPU execution that avoids inter-workgroup spin-waiting by using a hierarchical Blelloch scan to compute element positions, ensuring consistent behavior across heterogeneous GPU schedulers [S1, unofficial]. It sets itself apart from sibling methods like [Global Accumulator + Compaction](10410-global-accumulator-compaction.md) by eliminating cross-workgroup polling and busy-waits, which can degenerate into wasted cycles on devices without fixed scheduling guarantees [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** An array of $N$ elements with 32-bit integer keys.
*   **Output:** The same elements sorted by key.
*   **Auxiliary Storage:** Buffers for tile sums and intermediate scan results, sized $O(N/w_g)$ where $w_g$ is the workgroup size [S1, unofficial].

**Data Structures:**
*   **Indicator Stream:** A boolean or integer array indicating the presence of a specific digit value $d$ for each element.
*   **Tile Sums Array:** An array of length $N/w_g$ storing the sum of counts for each workgroup tile.
*   **Prefix Buffers:** Buffers storing the results of the upsweep and downsweep passes.

**Parameters:**
*   **Radix Size:** 256 (8 bits per pass) [S1, unofficial].
*   **Number of Passes:** 4 (for 32-bit keys) [S1, unofficial].
*   **Workgroup Size ($w_g$):** Not given in the sources.

**Algorithm Steps:**
The full 32-bit sort is performed by executing four consecutive passes of an 8-bit radix sort, starting from the least significant bits [S1, unofficial]. In the $k$-th pass ($k \in \{0, 1, 2, 3\}$), the algorithm processes bits $k \times 8$ to $(k + 1) \times 8 - 1$ of the key [S1, unofficial]. Each pass consists of three steps:

1.  **Local Histogram:** Each workgroup counts the frequencies of the current digit value $d$ within its local elements [S1, unofficial].
2.  **Prefix Scan:** A hierarchical Blelloch scan computes the global base offset $\text{base}_d$ and the inter-workgroup prefix $\text{prefix}_{wg}$ [S1, unofficial].
    *   **Forward Pass (Upsweep):**
        *   Level $l_0$: Performs an exclusive scan inside each workgroup tile of size $w_g$ over the indicator stream for digit $d$. This produces tile sums, forming an array of length $N/w_g$ [S1, unofficial].
        *   Level $l_1$: Runs the same Blelloch scan on the tile sums array to obtain prefixes across tiles. This recurses until a single level remains [S1, unofficial].
        *   $\text{base}_d$ is read from the last level as the sum of counts of digits smaller than $d$ [S1, unofficial].
    *   **Backward Pass (Downsweep):**
        *   The prefixes computed at level $l_1$ are added back to their corresponding $l_0$ tiles, and then to individual elements, yielding the inter-workgroup offset $\text{prefix}_{wg}$ [S1, unofficial].
3.  **Scatter:** Each element is written to its new position $\text{pos}$ in the sorted array [S1, unofficial].

**Formulas:**
The final position $\text{pos}$ for an element originally at index $i$ is calculated as [S1, unofficial]:
$$ \text{pos} = \text{base}_d + \text{rank}_i $$
where:
*   $d$ is the specific digit value of the element.
*   $\text{base}_d$ is the global offset equal to the total count of all elements with a digit value less than $d$.
*   $\text{rank}_i$ is the count of elements appearing before index $i$ sharing the same digit value $d$.

The rank $\text{rank}_i$ is decomposed into [S1, unofficial]:
$$ \text{rank}_i = \text{prefix}_{wg} + \text{rank}_{\text{local}_i} $$
where:
*   $\text{prefix}_{wg}$ is the inter-workgroup rank (total count of elements with digit $d$ in all workgroups processed before the current one).
*   $\text{rank}_{\text{local}_i}$ is the intra-workgroup rank (number of elements with digit $d$ within the current workgroup before $i$).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assume global arrays: keys[N], values[N], output_keys[N], output_values[N]
# Assume workgroup size wg, and helper functions for Blelloch scan

def wait_free_radix_sort(keys, values, N):
    # Perform 4 passes for 32-bit keys
    for k in range(4):
        # Extract digit d for each element
        digits = [(keys[i] >> (k * 8)) & 0xFF for i in range(N)]
        
        # Step 1: Local Histogram
        # Each workgroup computes local counts for each digit 0-255
        local_histograms = compute_local_histograms(keys, digits, N, wg)
        
        # Step 2: Prefix Scan (Hierarchical Blelloch)
        # Compute base_d and prefix_wg for each digit
        base_offsets, inter_workgroup_prefixes = hierarchical_blelloch_scan(local_histograms, N, wg)
        
        # Step 3: Scatter
        for i in range(N):
            d = digits[i]
            # Calculate rank_i
            # rank_local_i is computed during the downsweep of the scan
            # For simplicity, assume we have access to the local rank from the scan result
            rank_local_i = get_local_rank(i, d, local_histograms) 
            prefix_wg = inter_workgroup_prefixes[d]
            
            rank_i = prefix_wg + rank_local_i
            pos = base_offsets[d] + rank_i
            
            output_keys[pos] = keys[i]
            output_values[pos] = values[i]
        
        # Swap keys/values with output for next pass
        keys, values = output_keys, output_values

def hierarchical_blelloch_scan(local_histograms, N, wg):
    # Upsweep
    # Level l0: Exclusive scan within each workgroup
    tile_sums = []
    for wg_id in range(N // wg):
        # Scan the local histogram for this workgroup
        # Returns the sum of the histogram for each digit
        tile_sum = sum(local_histograms[wg_id])
        tile_sums.append(tile_sum)
    
    # Level l1+: Scan tile_sums recursively until single level
    # This provides global counts for all digits
    global_counts = blelloch_scan_recursive(tile_sums)
    
    # Compute base_d: sum of counts of digits smaller than d
    # This requires prefix sums over the global_counts array across digits
    base_offsets = compute_prefix_sum_over_digits(global_counts)
    
    # Downsweep
    # Add prefixes from l1 back to l0 tiles and then to elements
    # This yields the inter-workgroup offset prefix_wg for each element
    inter_workgroup_prefixes = compute_inter_workgroup_prefixes(local_histograms, global_counts, N, wg)
    
    return base_offsets, inter_workgroup_prefixes
```

**Edge Cases and Pitfalls:**
*   **Precision:** Not given in the sources.
*   **Boundaries:** The algorithm assumes $N$ is divisible by $w_g$ or handles remainder workgroups appropriately. Not given in the sources.
*   **Degenerate Input:** The algorithm handles any input distribution as it relies on histogram counts.

## Subtopics

### [8-bit Radix Sort](2599-8-bit-radix-sort.md)
This method performs a single pass of a radix sort on 64-bit keys by processing one 8-bit chunk per iteration, designed for GPU execution within a wait-free orchestration framework.

### [In-block shuffle and coalesced mapping](15056-in-block-shuffle-and-coalesced-mapping.md)
This method modifies the shuffle steps of radix sorting to achieve coalesced memory access, addressing the performance penalty of random-like scattering patterns in global memory writes. It sets itself apart from standard element-wise mapping by performing local shuffling in shared memory to cluster same-radix elements, followed by a coalesced cluster-wise mapping to global memory, ensuring that the $i$-th thread outputs data to the $i$-th position in a 128-byte aligned region.

## Operating characteristics
*   **Time Complexity:** Linear work $O(N)$ [S1, unofficial].
*   **Memory Complexity:** $O(N/w_g)$ auxiliary storage [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Avoids spin waiting across heterogeneous GPU schedulers; consistent behavior across devices; linear work complexity [S1, unofficial].
*   **Limitations:** Requires multiple compute dispatches for each level of the scan; may have higher overhead for small $N$ due to dispatch costs [S1, unofficial].
*   **When to use:** When running on heterogeneous GPU hardware where scheduling order is not guaranteed, and avoiding busy-waits is critical [S1, unofficial].
*   **When not to use:** When hardware-specific optimizations (like CUDA's warp-level primitives) are available and scheduling is deterministic [S1, unofficial].

## Background
Originated in the WebSplatter paper, which enables cross-device efficient Gaussian Splatting in Web Browsers via WebGPU [S1, unofficial]. The method was designed to address the issue of inter-workgroup spin-waiting in previous parallel sorting approaches, which assumed a fixed scheduling order and degenerated into busy-waits on devices without such guarantees [S1, unofficial].

## Sources
[S1] WebSplatter: Enabling Cross-Device Efficient Gaussian Splatting in Web Browsers via WebGPU, https://doi.org/10.48550/arxiv.2602.03207, fetched 2026-10-26 (unofficial)
