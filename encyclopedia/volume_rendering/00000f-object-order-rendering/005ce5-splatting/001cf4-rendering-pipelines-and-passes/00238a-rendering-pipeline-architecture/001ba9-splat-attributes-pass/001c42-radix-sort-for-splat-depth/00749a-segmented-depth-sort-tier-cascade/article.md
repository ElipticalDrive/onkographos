# Segmented Depth Sort (Tier Cascade)

Also called: Four-tier block radix sort cascade, Tier Cascade

## Overview
Segmented Depth Sort partitions a set of splats into contiguous segments defined by an offset array and sorts each segment independently based on a depth key [S1, unofficial]. It solves the problem of ordering per-macro-tile gaussian lists produced by binning, where the segment index implicitly encodes spatial location [S1, unofficial]. Unlike global composite-key sorts that simultaneously group by tile and order by depth, this method operates on 32-bit depth keys alone, halving radix sort memory traffic and reducing the number of radix passes [S1, unofficial]. It is implemented as a four-tier block radix sort cascade followed by a persistent merge phase for overflow segments, executed as fully asynchronous kernel launches on a single CUDA stream [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- A list of splats grouped by macro-tile. The input is logically partitioned into $M$ segments, where $M$ is the number of macro-tiles.
- For each segment $i \in [0, M-1]$, an offset array defines the start and end indices in the global buffer.
- The sort key is the 32-bit IEEE 754 float bit pattern of the depth value, reinterpreted as an unsigned integer to preserve ordering [S1, unofficial].

**Outputs:**
- A globally sorted list of splats, where splats within each macro-tile segment are ordered by increasing depth.

### Data Structures
**Segment Offsets:**
- An array of size $2M$ (or $M+1$ prefix sums) storing the start and end indices for each macro-tile segment in the global splat buffer.
- Produced by a prefix sum over per-macro-tile gaussian counts [S1, unofficial].

**Overflow Descriptors:**
- Emitted by Tier 3 (T3) for segments exceeding 8,192 items.
- Contains metadata required to split the segment into 8,192-item blocks and locate them in device memory [S1, unofficial].

**Merge Tree Metadata:**
- A device-side binary merge tree structure.
- Leaf nodes represent sorted 8,192-item runs.
- Internal nodes represent merge partitions, with run size doubling at each level [S1, unofficial].

**FIFO Work Queue:**
- A device-side queue storing pending merge partitions.
- Managed via atomic counters for CTA claiming and enqueueing [S1, unofficial].

### Algorithm Steps
1.  **Macro-tile Binning:**
    -   Perform conservative per-row ellipse-contour scan at macro-tile granularity to determine visibility [S1, unofficial].
    -   Compute per-macro-tile gaussian counts.
    -   Execute a two-level prefix sum over the histogram to produce global write offsets and segment boundaries [S1, unofficial].

2.  **Fill Pass:**
    -   Replay per-batch intersection using visibility bitmasks.
    -   Emit $(depth, gaussian-id)$ records into the global buffer at precomputed offsets.
    -   Depth is stored as raw IEEE 754 float bits, exploiting the property that positive floats sort correctly under unsigned integer comparison [S1, unofficial].
    -   Output size is $8 \times M$ bytes per macro-tile pair (4-byte depth key + 4-byte gaussian-id) [S1, unofficial].

3.  **Tier Cascade Sort:**
    -   Launch four tiers (T0–T3) asynchronously. Each tier uses CUB's block radix ranking primitives with four 8-bit radix passes [S1, unofficial].
    -   Launch one CUDA Thread Block (CTA) per segment.
    -   Each CTA checks if its segment size falls within the tier's capacity range. If yes, it sorts the segment; otherwise, it early-returns [S1, unofficial].
    -   **Tier Configuration:**
        -   **T0:** CTA Size 64, Capacity $\le 1,024$. Handles the majority of segments [S1, unofficial].
        -   **T1:** CTA Size 128, Capacity $1,025–2,048$ [S1, unofficial].
        -   **T2:** CTA Size 256, Capacity $2,049–4,096$ [S1, unofficial].
        -   **T3:** CTA Size 512, Capacity $4,097–8,192$. Emits overflow descriptors for segments exceeding 8,192 items [S1, unofficial].
    -   Use Programmatic Dependent Launch (PDL, SM $\ge$ 9.0) to overlap adjacent tiers, avoiding full kernel-completion barriers between independent size classes [S1, unofficial].

4.  **Overflow Path (for segments > 8,192 items):**
    -   **T3 Epilog:** Splits overflow segments into 8,192-item blocks and emits overflow descriptors to device memory [S1, unofficial].
    -   **Persistent Block-Sort:** A persistent kernel claims overflow blocks via an atomic work counter and sorts them [S1, unofficial].

5.  **Merge Phase:**
    -   **Merge Tree Construction:** Construct a binary merge tree device-side. Leaf nodes merge pairs of adjacent 8,192-item runs; internal nodes double run size [S1, unofficial].
    -   **Persistent Merge Kernel:** Processes the merge tree through a device-side FIFO work queue. CTAs claim merge partitions via atomic operations and enqueue parent node partitions when children finish [S1, unofficial].
    -   **GPU Merge-Path Algorithm:** Each merge partition uses CTA-level partitioning via warp-cooperative binary search to determine balanced sub-problems, merged sequentially in shared memory [S1, unofficial].

6.  **Asynchronous Coordination:**
    -   Inter-kernel coordination (overflow descriptors, merge-tree metadata, FIFO queue state) is communicated entirely through device-side atomic counters and memory fences [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
RADIX_WIDTH = 8
NUM_RADIX_PASSES = 4  # 32 bits / 8 bits
MAX_TIER_CAPACITY = 8192
BLOCK_SIZE = 8192

# Data Structures
# segment_offsets: Array of size M+1, defining start/end indices for each macro-tile segment
# splat_buffer: Array of (depth_u32, gaussian_id) pairs, size N
# overflow_descriptors: List of (segment_start, segment_end) for overflow segments
# merge_tree: Device-side structure representing the binary merge tree of sorted blocks

def segmented_depth_sort(segment_offsets, splat_buffer):
    M = len(segment_offsets) - 1
    
    # 1. Tier Cascade Launch
    # Launch T0, T1, T2, T3 asynchronously. 
    # Each tier launches M CTAs (one per segment).
    
    # T0: Capacity <= 1024
    launch_tier(T0, segment_offsets, splat_buffer, capacity_range=(0, 1024))
    
    # T1: Capacity 1025-2048
    launch_tier(T1, segment_offsets, splat_buffer, capacity_range=(1025, 2048))
    
    # T2: Capacity 2049-4096
    launch_tier(T2, segment_offsets, splat_buffer, capacity_range=(2049, 4096))
    
    # T3: Capacity 4097-8192
    overflow_descriptors = launch_tier(T3, segment_offsets, splat_buffer, capacity_range=(4097, 8192))
    
    # 2. Overflow Handling
    if overflow_descriptors is not empty:
        # T3 Epilog already emitted descriptors. 
        # Launch Persistent Block-Sort for overflow blocks
        launch_persistent_block_sort(overflow_descriptors, splat_buffer, block_size=BLOCK_SIZE)
        
        # Construct Merge Tree for overflow blocks
        merge_tree = construct_merge_tree(overflow_descriptors, splat_buffer)
        
        # Launch Persistent Merge Kernel
        launch_persistent_merge_kernel(merge_tree, splat_buffer)

    # Wait for all asynchronous kernels to complete before rasterization consumes results
    synchronize_all_tiers_and_overflow()

def launch_tier(tier, segment_offsets, splat_buffer, capacity_range):
    M = len(segment_offsets) - 1
    # Launch M CTAs, one per segment
    # Use PDL if available to overlap with adjacent tiers
    
    for i in range(M):
        start = segment_offsets[i]
        end = segment_offsets[i+1]
        segment_size = end - start
        
        if capacity_range[0] <= segment_size <= capacity_range[1]:
            # Sort this segment using CUB block radix ranking
            # 4 passes of 8-bit radix sort on depth keys
            cub_block_radix_sort(splat_buffer[start:end], tier.cta_size)
            
            # If tier is T3 and segment_size > 8192 (should not happen due to range, 
            # but per notes T3 emits overflow for >8192, so range might be inclusive of upper bound 
            # or T3 handles up to 8192 and emits overflow for larger? 
            # Notes say T3 Capacity 4097-8192, and "Segments exceeding 8,192 items are handled by an overflow path. 
            # The T3 epilog splits each overflow segment...". 
            # This implies T3 handles up to 8192. Segments > 8192 are handled by the overflow path.
            # The tier launch logic likely checks size. If size > 8192, it goes to overflow path.
            # The table says T3 Capacity 4097-8192. So segments > 8192 are NOT sorted by T3.
            pass
        elif segment_size > MAX_TIER_CAPACITY:
            # This segment is too large for T3. 
            # Emit overflow descriptor.
            # Note: The tier launch itself might not sort it.
            # The overflow path handles it.
            # We need to capture this for the overflow path.
            pass
            
    # Return overflow descriptors if T3
    if tier == T3:
        return collect_overflow_descriptors()
    return None

def collect_overflow_descriptors():
    # Collect segments that exceeded 8192 items
    # This is done during the tier launch phase by checking segment sizes
    # and recording their offsets
    pass

def launch_persistent_block_sort(overflow_descriptors, splat_buffer, block_size):
    # Persistent kernel claims blocks from overflow_descriptors via atomic counter
    # Sorts each block of size block_size
    while True:
        block_idx = atomic_add(work_counter, 1)
        if block_idx >= len(overflow_descriptors):
            break
        start, end = overflow_descriptors[block_idx]
        # Split segment into blocks of size BLOCK_SIZE
        for offset in range(start, end, BLOCK_SIZE):
            end_block = min(offset + BLOCK_SIZE, end)
            cub_block_radix_sort(splat_buffer[offset:end_block], BLOCK_SIZE)

def construct_merge_tree(overflow_descriptors, splat_buffer):
    # Leaf nodes are the sorted blocks from overflow_descriptors
    # Build binary tree where internal nodes merge pairs of adjacent runs
    # Return tree structure for merge kernel
    pass

def launch_persistent_merge_kernel(merge_tree, splat_buffer):
    # Persistent kernel processes merge tree via FIFO queue
    while True:
        partition = atomic_claim_from_fifo()
        if partition is None:
            break
        # Use GPU Merge-Path Algorithm
        # Warp-cooperative binary search to determine balanced sub-problems
        # Merge sequentially in shared memory
        gpu_merge_path_merge(splat_buffer, partition)
        # Enqueue parent node partition when children finish
        enqueue_parent_partition(partition)
```

### Parameters
-   **Radix Width ($w$):** The bit width used per radix pass. Determines the number of passes required for the sort. Typically 8 bits [S1, unofficial].
-   **CTA Sizes:**
    -   T0: 64 threads
    -   T1: 128 threads
    -   T2: 256 threads
    -   T3: 512 threads [S1, unofficial]
-   **Batch Size:** Gaussians are processed in fixed-size partitioning batches of $B$ gaussians ($B=8,192$ in implementation) [S1, unofficial].

### Edge Cases and Pitfalls
-   **Precision:** Depth is stored as raw IEEE 754 float bits. This relies on the property that positive floats sort correctly under unsigned integer comparison. Negative depths or NaNs may require special handling [S1, unofficial].
-   **Overflow:** Segments exceeding 8,192 items require the overflow path. Failure to handle overflow correctly results in unsorted data [S1, unofficial].
-   **Synchronization:** The pipeline is fully asynchronous. Host-device synchronization is only required before rasterization consumes the sorted results [S1, unofficial].
-   **Hardware Requirements:** Programmatic Dependent Launch (PDL) requires SM $\ge$ 9.0. Without PDL, full kernel-completion barriers are needed between tiers, increasing latency [S1, unofficial].

## Operating characteristics
-   **Time Complexity:** The sort is resolution-stable. At 4K resolution, the cascade sort remains below 0.08 ms across all benchmark scenes, compared to 2.5–4.2 ms for a global radix sort over the same content [S1, unofficial].
-   **Memory Complexity:** Memory traffic per pass is halved compared to global composite-key sorts because the key width is reduced from 64 bits to 32 bits [S1, unofficial].
-   **Strengths:**
    -   Significant performance improvement over global sorts, especially at high resolutions [S1, unofficial].
    -   Asynchronous execution hides latency [S1, unofficial].
    -   Efficient handling of variable segment sizes via tiered approach [S1, unofficial].
-   **Limitations:**
    -   Requires SM $\ge$ 9.0 for optimal performance (PDL) [S1, unofficial].
    -   Complex implementation due to multiple tiers and overflow handling [S1, unofficial].
-   **When to Use:** Use when macro-tile binning has already grouped splats by tile, and high-performance sorting is required for real-time rendering [S1, unofficial].
-   **When Not to Use:** Use global sort if splats are not pre-binned by tile or if hardware does not support PDL and latency is critical [S1, unofficial].

## Background
Segmented Depth Sort is part of the HiGS (Hierarchical Rendering Architecture for Real-Time 3D Gaussian Splatting) system [S1, unofficial]. It was introduced to address the performance bottlenecks of global sorting methods used in standard 3D Gaussian Splatting (3DGS) and gsplat [S1, unofficial]. The method leverages the hierarchical structure of macro-tiles to reduce the problem size and key width, enabling faster sorting [S1, unofficial].

## Sources
[S1] HiGS: A Hierarchical Rendering Architecture for Real-Time 3D Gaussian Splatting, https://doi.org/10.48550/arXiv.2606.00352, fetched 2026-10-29 (unofficial)
