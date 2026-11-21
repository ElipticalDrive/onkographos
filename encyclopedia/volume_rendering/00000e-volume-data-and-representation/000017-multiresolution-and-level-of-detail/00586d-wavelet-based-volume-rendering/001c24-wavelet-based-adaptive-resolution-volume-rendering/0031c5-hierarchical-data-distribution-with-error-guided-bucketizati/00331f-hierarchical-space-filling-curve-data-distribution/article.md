# Hierarchical Space-Filling Curve Data Distribution

Also called: Hilbert curve distribution, Space-filling curve traversal

## Overview
A data distribution scheme for parallel volume rendering that partitions wavelet-based volume blocks among processors to balance workload based on spatial and temporal reconstruction errors [S1]. It differs from sibling methods by combining a hierarchical space-filling curve traversal with error-guided bucketization, ensuring that blocks with similar error profiles are distributed to different processors to prevent contention during rendering [S1].

## How it works
**Inputs:**
*   WTSP tree structure containing wavelet-based volume blocks [S1].
*   Spatial error range and temporal error range of the tree nodes [S1].
*   Number of processors $P$ [S1].
*   Parameters $k_o$ (spatial levels per distribution unit) and $k_t$ (temporal levels per distribution unit) [S1].

**Outputs:**
*   Assignment of distribution units to processors [S1].

**Data Structures:**
*   **Distribution Unit:** A group of volume blocks spanning $k_o$ spatial levels and $k_t$ temporal levels [S1].
*   **Bucket:** A container for distribution units grouped by their maximum spatial and temporal error intervals [S1].
*   **Processor List:** Array of $P$ processors for round-robin assignment [S1].

**Parameters:**
*   $k_o$: Number of spatial levels spanned by each distribution unit. Typical value: 2 [S1].
*   $k_t$: Number of temporal levels spanned by each distribution unit. Typical value: 2 [S1].
*   Space-filling curve type: The specific curve used for traversal. Example: Hilbert curve [S1].

**Steps:**
1.  **Partition Error Ranges:** Divide the spatial error range and temporal error range of the WTSP tree nodes into discrete intervals [S1].
2.  **Create Buckets:** Iterate through all combinations of spatial intervals $i \in I_s$ and temporal intervals $j \in I_t$ to create a bucket $B_{i,j}$ for each spatial-temporal error interval combination [S1].
3.  **Traverse and Assign:** Traverse the WTSP tree along hierarchical space-filling curves [S1]. For every distribution unit encountered, calculate the maximum spatial error $e_s$ and maximum temporal error $e_t$ of all nodes within the unit [S1]. Identify the bucket $B_{i,j}$ such that $e_s \in i$ and $e_t \in j$, and append the unit to $B_{i,j}$ [S1]. The order of units within the bucket is determined by the space-filling curve traversal order [S1].
4.  **Adjust Buckets:** Adjust the boundaries of the spatial and temporal intervals so that each bucket holds a similar number of distribution units [S1].
5.  **Distribute:** Distribute all distribution units in each bucket among the $P$ processors in a round-robin fashion [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def distribute_data(wtsp_tree, num_processors, k_o, k_t, num_spatial_intervals, num_temporal_intervals, curve_type):
    # 1. Partition error ranges into discrete intervals
    spatial_intervals = partition_range(wtsp_tree.spatial_error_min, wtsp_tree.spatial_error_max, num_spatial_intervals)
    temporal_intervals = partition_range(wtsp_tree.temporal_error_min, wtsp_tree.temporal_error_max, num_temporal_intervals)

    # 2. Create buckets for all spatial-temporal interval combinations
    buckets = {(si, ti): [] for si in spatial_intervals for ti in temporal_intervals}

    # 3. Traverse and assign distribution units
    units = generate_distribution_units(wtsp_tree, k_o, k_t)
    ordered_units = traverse_with_curve(units, curve_type)

    for unit in ordered_units:
        max_s = max(node.spatial_error for node in unit.nodes)
        max_t = max(node.temporal_error for node in unit.nodes)
        si = find_interval(spatial_intervals, max_s)
        ti = find_interval(temporal_intervals, max_t)
        buckets[(si, ti)].append(unit)

    # 4. Adjust bucket intervals so each holds a similar number of units
    # The exact adjustment algorithm is not specified in the sources [G]
    spatial_intervals, temporal_intervals = adjust_bucket_boundaries(buckets, spatial_intervals, temporal_intervals)

    # 5. Distribute units in each bucket among processors round-robin
    processor_assignments = [[] for _ in range(num_processors)]
    for bucket_units in buckets.values():
        for idx, unit in enumerate(bucket_units):
            processor_assignments[idx % num_processors].append(unit)
    return processor_assignments

def partition_range(min_val, max_val, num_intervals):
    step = (max_val - min_val) / num_intervals
    return [(min_val + i * step, min_val + (i + 1) * step) for i in range(num_intervals)]

def find_interval(intervals, value):
    for interval in intervals:
        if interval[0] <= value < interval[1]:
            return interval
    return intervals[-1]

def generate_distribution_units(wtsp_tree, k_o, k_t):
    # Groups WTSP tree nodes into contiguous units spanning k_o spatial and k_t temporal levels
    # Implementation depends on specific tree layout [G]
    return [wtsp_tree.nodes[i:i + k_o * k_t] for i in range(0, len(wtsp_tree.nodes), k_o * k_t)]

def traverse_with_curve(units, curve_type):
    # Orders units according to the specified space-filling curve
    # Standard SFC traversal logic [G]
    return sorted(units, key=lambda u: compute_curve_index(u, curve_type))

def adjust_bucket_boundaries(buckets, spatial_intervals, temporal_intervals):
    # Iteratively adjusts interval boundaries to equalize bucket sizes
    # Exact method not provided in sources [G]
    return spatial_intervals, temporal_intervals
```

**Edge Cases and Pitfalls:**
*   **Empty Buckets:** If certain error combinations do not occur, buckets may remain empty, which must be handled during the adjustment step to avoid skewing load balancing [G].
*   **Boundary Conditions:** Nodes with maximum errors exactly on interval boundaries must be handled consistently (e.g., inclusive/exclusive bounds) to ensure deterministic assignment [G].

## Operating characteristics
*   **Time Complexity:** Reconstruction time for a data block is $O(c_1 \times k_o + c_2 \times k_o \times k_t)$, where $c_1$ is the time for an inverse 3D wavelet transform, $c_2$ is the time for an inverse 1D wavelet transform, $k_o$ is the number of levels in an octree node group, and $k_t$ is the number of levels in a time tree node group [S1].
*   **Memory Complexity:** Depends on the caching strategy (EVERY-K scheme), where data blocks are cached in main memory or swapped to disk if storage is insufficient [S1].
*   **Performance:** On a 32-processor PC cluster, using $k_o=2$ and $k_t=2$ with Hilbert curves provided effective load balancing for large datasets like RMI and SPOT [S1].
*   **Strengths:** Balances workload based on actual reconstruction error; efficient reconstruction due to limited wavelet transform depth [S1].
*   **Limitations:** Requires pre-computation of error ranges and bucket adjustments; dependent on the accuracy of error estimation [S1].

## Background
Originated in the context of parallel volume rendering for large datasets [S1]. The method builds on previous applications of space-filling curves in large data visualization and distributed client-server environments [S1]. It was developed to address load balancing issues in wavelet-based adaptive resolution volume rendering, specifically leveraging the WTSP (Wavelet-based Temporal-Spatial Pyramid) tree structure [S1].

## Sources
[S1] A Multiresolution Volume Rendering Framework for Large- ..., https://diglib.eg.org/bitstream/handle/10.2312/VG.VG05.011-019/011-019.pdf?sequence=1, fetched 2026-10-26
