# Tile-Based Comparison-Free Sorting

Also called: Comparison-Free Depth Sorting, Tile-Local Sorting

## Overview
Tile-Based Comparison-Free Sorting restricts depth sorting operations to individual tiles to avoid global synchronization, reduce hardware complexity, and achieve fixed latency per output [S1]. Unlike global radix sort, which incurs high memory traffic and variable latency due to histograms and prefix sums, this method provides deterministic O(N) cycles suitable for low-power hardware accelerators [S1]. It serves as the sorting mechanism for 3D Gaussian Splatting, ensuring Gaussians are ordered front-to-back for correct alpha compositing within each tile [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A list of $N$ Gaussian points, each with a projected depth value. The points are already assigned to tiles.
*   **Output:** The Gaussian points sorted front-to-back (increasing depth) within each tile.
*   **Sorting Key:** $\langle \text{tile-id}, \text{depth} \rangle$ [S1].

**Data Structures**
*   **Element Vector Table (EVT):** A bit vector of size $N$. Each bit corresponds to a Gaussian point. A value of `1` indicates the element is unsorted (available for selection); a value of `0` indicates it has been sorted and removed from the candidate set [S1].
*   **Depth Values:** $n$-bit fixed-point or floating-point representations of depth.

**Algorithm Steps**
The algorithm identifies the Largest Element (LE) iteratively $N$ times. In each iteration, the identified LE is marked as sorted in the EVT, and the process repeats until all elements are sorted [S1].

1.  **Initialization:** Initialize the EVT with all bits set to `1` [S1].
2.  **Concurrent Phase:** $k$ parallel clusters process partitions of the $N$ $n$-bit elements. Starting from the most significant bit (MSB), the clusters filter out smaller values through bitwise AND operations and multiplexer-based selection in cascaded blocks [S1].
    *   *Sign Bit Skipping:* Since near-plane culling excludes points with depth $< 0$, the sign bit is always 0. The operation on the sign bit is removed to reduce computation [S1].
3.  **Sequential Phase:** The outputs from the $k$ clusters pass to a linear array of $k$ cascaded blocks to further refine the selection and identify the single Largest Element (LE) [S1].
4.  **Data Conflict Resolution:** If duplicate depth values exist, multiple bits in the EVT might be selected as "largest." To resolve this and ensure deterministic selection, the algorithm computes a unique identifier for the first selected element using bitwise operations:
    $$FO_{NEW} = Fo \& (\sim Fo + 1'b1)$$
    where $Fo$ is the output mask from the previous stage [S1]. This isolates the least significant bit of the conflict set, ensuring a single element is chosen [S1].
5.  **Iteration:** The identified LE is removed from the EVT (bit set to `0`). The process repeats until the EVT contains all `0`s, signaling all $N$ elements are sorted [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tile_based_comparison_free_sort(gaussians, n_bits, k_clusters):
    """
    Sorts Gaussians within a tile using comparison-free logic.
    gaussians: List of depth values (n_bits wide).
    n_bits: Number of bits in depth representation.
    k_clusters: Number of parallel processing clusters.
    """
    N = len(gaussians)
    # EVT tracks unsorted elements; 1 means available, 0 means sorted
    evt = [1] * N 
    
    sorted_gaussians = []
    
    # Repeat N times to extract one element per iteration
    for _ in range(N):
        # Concurrent Phase: Find candidate largest elements
        # Simplified logic: In hardware, this involves bitwise reduction across n_bits
        # and k clusters. Here we simulate the identification of the max value.
        
        # Find the maximum depth value among unsorted elements
        max_depth = -float('inf')
        candidates = []
        
        for i in range(N):
            if evt[i] == 1:
                depth = gaussians[i]
                if depth > max_depth:
                    max_depth = depth
                    candidates = [i]
                elif depth == max_depth:
                    candidates.append(i)
        
        # Data Conflict Resolution: Select one candidate deterministically
        # Formula: FO_NEW = Fo & (~Fo + 1)
        # This isolates the least significant bit of the candidate mask
        if len(candidates) > 1:
            # Create a bitmask of candidates
            candidate_mask = 0
            for idx in candidates:
                candidate_mask |= (1 << idx)
            
            # Apply conflict resolution logic
            # Note: In hardware, this is bitwise. In Python simulation:
            # ~Fo is infinite bits, so we mask to N bits for clarity, 
            # but the logic holds for the LSB of the set.
            # FO_NEW = candidate_mask & (~candidate_mask + 1)
            # This effectively selects the lowest index in the tie.
            resolved_index = (candidate_mask & (~candidate_mask + 1)).bit_length() - 1
            
            # Map back to the actual index in the list
            # Since bit_length() - 1 gives the position of the LSB,
            # we need to find which candidate corresponds to that bit position.
            # However, the hardware logic ensures a unique output FO_NEW.
            # For this pseudocode, we pick the first candidate as the deterministic result 
            # equivalent to the LSB selection in the conflict set.
            selected_idx = candidates[0] 
        else:
            selected_idx = candidates[0]
            
        # Mark as sorted
        evt[selected_idx] = 0
        sorted_gaussians.append(gaussians[selected_idx])
        
    return sorted_gaussians
```

**Edge Cases and Pitfalls**
*   **Duplicate Depths:** The algorithm must handle duplicate depth values. Without conflict resolution, multiple elements might be identified as the "largest" simultaneously, leading to incorrect EVT updates [S1]. The bitwise resolution formula ensures a single element is selected per cycle [S1].
*   **Sign Bit:** The sign bit of the depth value is assumed to be 0 due to near-plane culling. If points with negative depth are not culled, the sign bit logic must be included, increasing complexity [S1].
*   **Variable N:** The number of Gaussians per tile varies. The hardware must handle this variability without pipeline stalls, which is achieved by the fixed-latency nature of the comparison-free logic [S1].

## Subtopics

### Parallel Comparison-Free Sorting
A tile-based, comparison-free sorting algorithm for Gaussian points that uses an Element Vector Table (EVT) to identify the largest element (LE) in O(N) deterministic cycles, avoiding the high memory overhead of radix sort [S1]. It is chosen when hardware resources are constrained and deterministic latency is required, as opposed to global sorting methods [S1].

## Operating characteristics
*   **Time Complexity:** O(N) deterministic cycles per tile, where N is the number of Gaussians in the tile [S1].
*   **Memory Complexity:** Low hardware complexity compared to radix sort, which requires histograms, prefix sums, and scatters [S1]. The EVT requires O(N) bits [S1].
*   **Performance:** Achieves full-HD rendering at 129 FPS, with 3× higher throughput and 5.98× smaller area compared to prior 3DGS accelerators [S1].
*   **Strengths:** Fixed per-output latency, no global synchronization, low hardware complexity, suitable for low-power AR/VR hardware [S1].
*   **Limitations:** Requires near-plane culling to skip sign bit logic; performance depends on the number of Gaussians per tile, which can vary significantly [S1].

## Background
The method is based on the parallel comparison-free algorithm described in [21, 22] (cited in [S1]), adapted for 3D Gaussian Splatting rendering pipelines. It addresses the bottleneck of global radix sort, which is resource-intensive and causes latency spikes due to variable latency and high memory traffic [S1]. The approach was implemented in a hardware accelerator design described in [S1], achieving significant improvements in throughput and area efficiency over previous designs like GSCore [10].

## Sources
[S1] A 129 FPS Full HD Real-Time Accelerator for 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2026.3683714, fetched 2026-10-27
