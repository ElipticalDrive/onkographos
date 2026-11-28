# Wait-Free Radix Sort

Also called: Hierarchical Blelloch Scan Sort

## Overview
A parallel sorting algorithm designed for GPU execution that avoids inter-workgroup spin-waiting by using a hierarchical Blelloch scan to compute element positions, ensuring consistent behavior across heterogeneous GPU schedulers [S1, unofficial]. It sets itself apart from sibling methods like [Global Accumulator + Compaction](10410-global-accumulator-compaction.md) by eliminating cross-workgroup polling and busy-waits, which can degenerate into wasted cycles on devices without fixed scheduling guarantees [S1, unofficial].

## How it works
**Inputs and Outputs:**
The method takes an array of $N$ elements with 32-bit integer keys as input and produces the same elements sorted by key as output [S1, unofficial]. It requires auxiliary storage buffers for tile sums and intermediate scan results, sized $O(N/w_g)$ where $w_g$ is the workgroup size [S1, unofficial].

**Data Structures:**
The algorithm maintains an indicator stream representing the presence of a specific digit value $d$ for each element, a tile sums array of length $N/w_g$ storing the sum of counts for each workgroup tile, and prefix buffers storing the results of the upsweep and downsweep passes [S1, unofficial].

**Parameters:**
The radix size is fixed at 256 (8 bits per pass) [S1, unofficial]. A full 32-bit sort requires exactly 4 consecutive passes [S1, unofficial]. The workgroup size $w_g$ is not given in the sources.

**Algorithm Steps:**
The full 32-bit sort is performed by executing four consecutive passes of an 8-bit radix sort, starting from the least significant bits [S1, unofficial]. In the $k$-th pass ($k \in \{0, 1, 2, 3\}$), the algorithm processes bits $k \times 8$ to $(k + 1) \times 8 - 1$ of the key [S1, unofficial]. Each pass consists of three sequential steps:

1. **Local Histogram:** Each workgroup counts the frequencies of the current digit value $d$ within its local elements [S1, unofficial].
2. **Prefix Scan:** A hierarchical Blelloch scan processes the histograms to compute the global base offset $\text{base}_d$ and the inter-workgroup prefix $\text{prefix}_{wg}$ [S1, unofficial].
    * **Forward Pass (Upsweep):** Level $l_0$ performs an exclusive scan inside each workgroup tile of size $w_g$ over the indicator stream for digit $d$, producing tile sums that form an array of length $N/w_g$ [S1, unofficial]. Level $l_1$ runs the same Blelloch scan on the tile sums array to obtain prefixes across tiles, and the process recurses until a single level remains [S1, unofficial]. $\text{base}_d$ is read from the last level as the sum of counts of digits smaller than $d$ [S1, unofficial].
    * **Backward Pass (Downsweep):** The prefixes computed at level $l_1$ are added back to their corresponding $l_0$ tiles, and then to individual elements, yielding the inter-workgroup offset $\text{prefix}_{wg}$ [S1, unofficial]. Each level is executed as an independent compute dispatch consuming the previous level's buffers, with only intra-workgroup barriers inside a tile [S1, unofficial].
3. **Scatter:** Each element is written to its new position $\text{pos}$ in the sorted array [S1, unofficial].

**Formulas:**
The final position $\text{pos}$ for an element originally at index $i$ is calculated as [S1, unofficial]:
$$ \text{pos} = \text{base}_d + \text{rank}_i $$
where $d$ is the specific digit value of the element, $\text{base}_d$ is the global offset equal to the total count of all elements with a digit value less than $d$, and $\text{rank}_i$ is the count of elements appearing before index $i$ sharing the same digit value $d$ [S1, unofficial].

The rank $\text{rank}_i$ is decomposed into [S1, unofficial]:
$$ \text{rank}_i = \text{prefix}_{wg} + \text{rank}_{\text{local}_i} $$
where $\text{prefix}_{wg}$ is the inter-workgroup rank (total count of elements with digit $d$ in all workgroups processed before the current one) and $\text{rank}_{\text{local}_i}$ is the intra-workgroup rank (number of elements with digit $d$ within the current workgroup before $i$) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def wait_free_radix_sort(keys, values, N, wg):
    # 4 passes for 32-bit keys, processing LSB to MSB
    for k in range(4):
        # Extract 8-bit digit for each element
        digits = [(keys[i] >> (k * 8)) & 0xFF for i in range(N)]
        
        # Step 1: Local Histogram
        # Each workgroup counts frequencies of digit d in its local elements
        local_histograms = [[0] * 256 for _ in range(N // wg)]
        for i in range(N):
            local_histograms[i // wg][digits[i]] += 1
        
        # Step 2: Prefix Scan (Hierarchical Blelloch)
        # Forward pass (upsweep)
        tile_sums = [sum(hist) for hist in local_histograms]
        # Recursively scan tile sums until single level remains
        global_counts = blelloch_upsweep(tile_sums)
        # base_d is sum of counts of digits smaller than d
        base_offsets = [sum(global_counts[:d]) for d in range(256)]
        
        # Backward pass (downsweep)
        # Add prefixes from higher levels back to l0 tiles and elements
        inter_workgroup_prefixes = blelloch_downsweep(local_histograms, global_counts, N, wg)
        
        # Step 3: Scatter
        for i in range(N):
            d = digits[i]
            wg_id = i // wg
            local_idx = i % wg
            # rank_local_i: count of digit d in current WG before index i
            rank_local_i = sum(1 for j in range(local_idx) if digits[j] == d)
            prefix_wg = inter_workgroup_prefixes[wg_id][d]
            rank_i = prefix_wg + rank_local_i
            pos = base_offsets[d] + rank_i
            output_keys[pos] = keys[i]
            output_values[pos] = values[i]
        
        # Swap buffers for next pass
        keys, values = output_keys, output_values

def blelloch_upsweep(arr):
    # Standard recursive Blelloch upsweep: pairwise reduction
    if len(arr) == 1: return arr
    tile_sums = []
    for i in range(0, len(arr), 2):
        tile_sums.append(arr[i] + arr[i+1])
    return blelloch_upsweep(tile_sums)

def blelloch_downsweep(local_histograms, global_counts, N, wg):
    # Standard Blelloch downsweep: propagate prefixes back to tiles and elements
    inter_workgroup_prefixes = [[0] * 256 for _ in range(N // wg)]
    for wg_id in range(N // wg):
        for d in range(256):
            # prefix_wg is the cumulative sum of tile sums for workgroups before this one
            prefix_wg = sum(sum(local_histograms[j][d]) for j in range(wg_id))
            inter_workgroup_prefixes[wg_id][d] = prefix_wg
    return inter_workgroup_prefixes
```

**Edge Cases and Pitfalls:**
* **Precision:** Not given in the sources.
* **Boundaries:** The algorithm assumes $N$ is divisible by $w_g$ or handles remainder workgroups appropriately [G].
* **Degenerate Input:** The algorithm handles any input distribution as it relies on histogram counts [S1, unofficial].

## Subtopics

### [8-bit Radix Sort](2599-8-bit-radix-sort.md)
This method performs a single pass of a radix sort on 64-bit keys by processing one 8-bit chunk per iteration, designed for GPU execution within a wait-free orchestration framework[3368-subgroup-based-radix-sort-partition.md][10404-radix-sort-gpu.md]. It is designed for WebGPU and Vulkan compute shaders, ensuring consistent behavior across heterogeneous GPU schedulers by avoiding race conditions inherent in atomic-based sorting. Unlike [Subgroup-based Radix Sort Partition](3368-subgroup-based-radix-sort-partition.md) which minimizes atomic contention via subgroup ballots, or [Radix Sort (GPU)](10404-radix-sort-gpu.md) which separates counting and scattering into distinct shader passes, this method integrates the scan and scatter within a ping-pong buffer architecture to handle arbitrary data types and key spaces[3368-subgroup-based-radix-sort-partition.md][10404-radix-sort-gpu.md].

### [In-block shuffle and coalesced mapping](15056-in-block-shuffle-and-coalesced-mapping.md)
This method modifies the shuffle steps of radix sorting to achieve coalesced memory access, addressing the performance penalty of random-like scattering patterns in global memory writes. It sets itself apart from standard element-wise mapping by performing local shuffling in shared memory to cluster same-radix elements, followed by a coalesced cluster-wise mapping to global memory, ensuring that the $i$-th thread outputs data to the $i$-th position in a 128-byte aligned region.

## Operating characteristics
* **Time Complexity:** Linear work $O(N)$ [S1, unofficial].
* **Memory Complexity:** $O(N/w_g)$ auxiliary storage [S1, unofficial].
* **Measured Performance:** Not given in the sources.
* **Scaling:** Not given in the sources.
* **Strengths:** Avoids spin waiting across heterogeneous GPU schedulers; consistent behavior across devices; linear work complexity [S1, unofficial].
* **Limitations:** Requires multiple compute dispatches for each level of the scan; may have higher overhead for small $N$ due to dispatch costs [S1, unofficial].
* **When to use:** When running on heterogeneous GPU hardware where scheduling order is not guaranteed, and avoiding busy-waits is critical [S1, unofficial].
* **When not to use:** When hardware-specific optimizations (like CUDA's warp-level primitives) are available and scheduling is deterministic [S1, unofficial].

## Background
Introduced in the WebSplatter paper for depth sorting within a Gaussian Splatting rasterization pipeline [S1, unofficial]. The method addresses the issue of inter-workgroup spin-waiting in prior parallel sorting approaches that assumed a fixed scheduling order and degenerated into busy-waits on devices without such guarantees [S1, unofficial].

## Sources
[S1] WebSplatter: Enabling Cross-Device Efficient Gaussian Splatting in Web Browsers via WebGPU, https://doi.org/10.48550/arxiv.2602.03207, fetched 2026-10-26 (unofficial)
