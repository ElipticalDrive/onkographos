# Wait-Free Radix Sort

Also called: Hierarchical Blelloch Scan Sort

## Overview
A parallel sorting algorithm designed for heterogeneous GPU environments (specifically WebGPU) that avoids inter-workgroup spin-waiting by using a hierarchical Blelloch scan to compute element positions [S1, unofficial]. It solves the problem of stable, wait-free sorting across workgroups where scheduling order is not guaranteed, ensuring consistent behavior without busy-waiting [S1, unofficial]. It sets itself apart from siblings like [CUDA-based Exclusive Prefix Sum for Vertex Indexing](1112-cuda-based-exclusive-prefix-sum-for-vertex-indexing.md) by explicitly decoupling workgroup synchronization via a multi-level scan dispatch rather than relying on fixed scheduling assumptions or tile-based rasterizer constraints [S1, unofficial].

## How it works
**Inputs:** An array of $N$ keys (e.g., 32-bit depth values) to be sorted.
**Outputs:** A permutation of the input array sorted by key.

**Data Structures:**
*   `input`: Array of $N$ keys.
*   `output`: Array of $N$ values (or indices) to be reordered.
*   `histogram`: Local buffer per workgroup of size 256 (for 8-bit radix), storing counts of each digit value $d \in [0, 255]$.
*   `tile_sums`: Array of size $N/wg$, storing the sum of each workgroup's histogram for a specific digit.
*   `prefixes`: Array of size $N/wg$, storing the exclusive prefix sums of `tile_sums`.
*   `base_d`: Scalar value representing the global offset for digit $d$.

**Parameters:**
*   $N$: Number of elements.
*   $wg$: Workgroup size (tile size).
*   $B$: Number of bits per pass (8).
*   $R$: Radix size ($2^B = 256$).
*   $P$: Number of passes ($32/B = 4$).

**Algorithm Steps:**
The full sort is performed by executing $P$ consecutive passes of an 8-bit radix sort, starting from the least significant bits [S1, unofficial]. In the $k$-th pass ($k=0 \dots P-1$), the algorithm processes bits $k \times 8$ to $(k + 1) \times 8 - 1$ of the key [S1, unofficial]. Each pass consists of three steps:

1.  **Local Histogram:** Each workgroup $w$ computes a local histogram of the digit values extracted from its assigned elements. Let $d_i$ be the digit value of element $i$. The workgroup increments `histogram[d_i]` for each element.
2.  **Prefix Scan:** A hierarchical Blelloch scan is executed to compute two values:
    *   $\text{base}_d$: The global offset for digit $d$, equal to the total count of all elements with a digit value less than $d$ across all workgroups.
    *   $\text{prefix}_{wg}$: The inter-workgroup rank, representing the total count of elements with digit $d$ in all workgroups processed before the current workgroup $w$.
    
    The Blelloch scan is implemented as a two-phase process:
    *   **Forward Pass (Upsweep):** Level $l_0$ performs an exclusive scan inside each workgroup tile of size $wg$ over the indicator stream for digit $d$, producing tile sums forming an array of length $N/wg$ [S1, unofficial]. Level $l_1$ runs the same Blelloch scan on this array to obtain prefixes across tiles, recursing until a single level remains [S1, unofficial].
    *   **Backward Pass (Downsweep):** The prefixes computed at the higher levels are added back to their corresponding $l_0$ tiles, and then to individual elements, yielding the intra-workgroup ranks and the inter-workgroup offset $\text{prefix}_{wg}$ [S1, unofficial].
    *   $\text{base}_d$ is read from the last level of the upsweep as the sum of counts of digits smaller than $d$ [S1, unofficial].
3.  **Scatter:** Each element $i$ is written to its new position in the output array. The position is calculated as:
    $$pos = \text{base}_d + \text{rank}_i$$
    where:
    $$\text{rank}_i = \text{prefix}_{wg} + \text{rank}_{\text{local}_i}$$
    Here, $\text{rank}_{\text{local}_i}$ is the intra-workgroup rank (number of elements with digit $d$ within the current workgroup before index $i$) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def wait_free_radix_sort(input_keys, N, wg):
    output = [None] * N
    # 32-bit sort requires 4 passes of 8-bit radix sort
    for k in range(4):
        shift = k * 8
        # Extract digit d for each element
        digits = [(key >> shift) & 0xFF for key in input_keys]
        
        # Step 1: Local Histogram
        # Each workgroup computes its local histogram
        # This is done in parallel by workgroups
        local_histograms = compute_local_histograms(digits, N, wg)
        
        # Step 2: Prefix Scan (Hierarchical Blelloch)
        # Compute tile sums from local histograms for the current digit d
        # Note: The scan is performed for each digit d independently or 
        # aggregated. The source implies computing offsets for the digit 
        # being processed. We assume the scan operates on the counts of 
        # the specific digit d across workgroups.
        
        # Extract counts for digit d from each workgroup's histogram
        # tile_counts[d] = sum of counts of digit d in each workgroup
        tile_counts = [hist[d] for hist in local_histograms]
        
        # Perform Blelloch Scan on tile_counts to get prefixes
        # Returns array of length N/wg where prefix[i] is sum of tile_counts[0..i-1]
        prefixes = blelloch_scan(tile_counts)
        
        # base_d is the total sum of counts of digits < d
        # This requires running the scan for all digits or accumulating sums.
        # Based on source: "read base_d from the last level as the sum of counts 
        # of digits smaller than d".
        # We assume base_d is precomputed or computed via a similar scan over 
        # the cumulative sums of all digit counts.
        base_d = compute_base_offset(d, local_histograms, N, wg)
        
        # Step 3: Scatter
        for i in range(N):
            d_i = digits[i]
            w = i // wg
            idx_in_wg = i % wg
            
            # prefix_wg is the prefix sum for workgroup w
            prefix_wg = prefixes[w]
            
            # rank_local_i is the count of elements with digit d_i in the 
            # current workgroup before index i.
            # This requires a local scan within the workgroup.
            rank_local_i = compute_local_rank(i, d_i, local_histograms[w])
            
            rank_i = prefix_wg + rank_local_i
            pos = base_d + rank_i
            
            output[pos] = input_keys[i]
            
        # Update input_keys for next pass
        input_keys = output

def blelloch_scan(data):
    """
    Standard Blelloch Scan (Exclusive) on an array.
    Implemented as upsweep and downsweep.
    """
    n = len(data)
    if n == 1:
        return [0]
    
    # Upsweep (reduce)
    # Build a tree of sums
    tree = [0] * (2 * n)
    tree[n:] = data[:]
    
    level = n
    while level > 1:
        for i in range(0, level, 2):
            tree[i // 2] = tree[i] + tree[i + 1]
        level //= 2
    
    # Read base_d from the last level (root) if needed, 
    # but for exclusive scan we need prefixes.
    
    # Downsweep
    tree[n] = 0 # Clear the root for exclusive scan logic if needed, 
                # but standard Blelloch sets the root to 0 for exclusive scan?
                # Actually, for exclusive scan, the root of the downsweep 
                # is set to 0.
    
    # Re-initialize tree for downsweep
    # The upsweep computed the total sum at tree[0].
    # For exclusive scan, we set the root to 0.
    tree[0] = 0 
    
    level = 1
    while level < n:
        for i in range(level * 2 - 1, -1, -2):
            if i + 1 < 2 * n:
                tree[i + 1] += tree[i]
        level *= 2
        
    return tree[n:]

def compute_local_rank(i, d_i, local_hist):
    """
    Compute the number of elements with digit d_i in the current workgroup 
    before index i.
    """
    # This requires scanning the local elements.
    # In a real implementation, this is done via a local scan within the workgroup.
    # Here we simulate it by counting.
    count = 0
    # In practice, this is a local exclusive scan of the indicator vector
    # for digit d_i within the workgroup.
    return count # Placeholder for local scan logic
```

**Edge Cases and Pitfalls:**
*   **Precision:** The algorithm relies on integer arithmetic for counts. Ensure that the total count $N$ fits within the data type used for offsets.
*   **Boundaries:** The hierarchical scan must handle cases where $N$ is not a multiple of $wg$. Padding or boundary checks are required in the local histogram and scan steps.
*   **Degenerate Input:** If all keys are identical, the sort remains stable, and all elements are placed in their original relative order.

## Subtopics

### [8-bit Radix Sort](2599-8-bit-radix-sort.md)
The fundamental pass used in the wait-free radix sort. It sorts elements based on a single 8-bit digit of the key. It is chosen for its efficiency in parallel histogram computation and its compatibility with the hierarchical scan structure.

## Operating characteristics
*   **Time Complexity:** Linear work $O(N)$ per pass, with $P$ passes, resulting in $O(P \cdot N)$ total work. The hierarchical scan adds a logarithmic factor in terms of workgroup count, but the total work remains linear in $N$.
*   **Memory Complexity:** $O(N/wg)$ auxiliary storage for the tile sums and prefixes [S1, unofficial].
*   **Performance:** Avoids spin waiting across heterogeneous GPU schedulers, making it robust across different hardware [S1, unofficial].
*   **Strengths:** Wait-free, stable, works on WebGPU and heterogeneous devices.
*   **Limitations:** Requires multiple compute dispatches for the hierarchical scan levels, which may introduce overhead.

## Background
Originating from the WebSplatter project, this method was designed to enable efficient Gaussian Splatting in web browsers via WebGPU [S1, unofficial]. It addresses the limitations of CUDA-based approaches that assume fixed scheduling orders, which can degenerate into busy-waits on devices without such guarantees [S1, unofficial].

## Sources
[S1] WebSplatter: Enabling Cross-Device Efficient Gaussian Splatting in Web Browsers via WebGPU, https://doi.org/10.48550/arxiv.2602.03207, fetched 2026-10-26 (unofficial)
