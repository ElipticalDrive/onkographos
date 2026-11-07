# List-based Pre-classification

Also called: Bucket sort pre-classification, Iso-value index buffers, List-based Pre-classification Performance

## Overview

List-based pre-classification is an optimization for interactive transfer function changes in volume rendering that avoids rebuilding index buffers on the CPU by pre-sorting voxel indices into iso-value buckets, allowing fast pointer merging [S1, unofficial][S2]. Unlike naive pre-classification which requires rebuilding and reloading the index buffer on every transfer function change (causing CPU-GPU bandwidth bottlenecks), this method uses pre-sorted buffers and pointer merging for interactive rates [S1, unofficial][S2].

## How it works

**Inputs and Outputs:**
*   **Input:** A 3D volume dataset with density values; a traversal order (e.g., front-to-back slices); proxy geometry indices corresponding to voxels [S1, unofficial][S2].
*   **Output:** Pre-loaded iso-value index buffers in graphics hardware (video or AGP memory) and pointers to the start of each iso-value array in main memory [S1, unofficial][S2].

**Data Structures:**
1.  **Iso-value Buckets:** The range of possible density/iso-values is divided into 256 intervals [S1, unofficial][S2].
2.  **Index Buffers:** Arrays of vertex indices for proxy geometry (splat vertices) stored in graphics hardware (video memory or AGP memory) [S1, unofficial][S2]. The placement in AGP memory does not affect performance [S1, unofficial][S2].
3.  **Pointers:** An array in main memory where each entry points to the start of the index list for a specific iso-value interval [S1, unofficial][S2].

**Steps:**

1.  **Bucket Sort Voxels:** Bucket sort the voxels based on their density values [S1, unofficial][S2].
2.  **Compute Iso-value Lists:** Compute iso-value lists for each slice of each traversal order [S1, unofficial][S2]. This differs from prior work (e.g., Mueller et al. [16]) which built iso-value lists for the whole volume [S1, unofficial].
3.  **Sort and Rearrange Indices:** Sort the indices of the corresponding proxy geometry accordingly and rearrange them into iso-value index buffers [S1, unofficial][S2].
4.  **Pre-load Buffers:** Pre-load the index buffers into graphics hardware (video memory or AGP memory) before interactive rendering [S1, unofficial][S2].
5.  **Maintain Pointers:** Maintain pointers to the start of each iso-value array in main memory [S1, unofficial][S2].
6.  **Interactive Update:** When the transfer function changes, collect appropriate pointers and merge them quickly to send visible voxels to the rendering pipeline [S1, unofficial][S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_list_based_preclassification(volume_data, traversal_slices):
    num_buckets = 256
    bucket_lists = [[] for _ in range(num_buckets)]
    
    for slice_data in traversal_slices:
        for voxel_index, density in enumerate(slice_data):
            bucket_id = int(density * 255)
            bucket_id = max(0, min(bucket_id, num_buckets - 1))
            bucket_lists[bucket_id].append(voxel_index)
            
    pointers = [0] * num_buckets
    current_offset = 0
    for i in range(num_buckets):
        pointers[i] = current_offset
        current_offset += len(bucket_lists[i])
        
    return bucket_lists, pointers

def update_for_transfer_function(pointers, bucket_lists, visible_buckets):
    merged_indices = []
    for bucket_id in visible_buckets:
        start_ptr = pointers[bucket_id]
        merged_indices.extend(bucket_lists[bucket_id])
    return merged_indices
```

**Parameters:**
*   **Number of Iso-value Intervals:** 256 [S1, unofficial][S2].
*   **Storage Location:** Video memory or AGP memory, depending on size [S1, unofficial][S2].

**Edge Cases and Pitfalls:**
*   **Unchanged Transfer Function:** When the transfer function is unchanged, list-based pre-classification is slightly slower than traditional pre-classification [S1, unofficial][S2].
*   **Memory Placement:** Putting index buffers in AGP memory does not affect performance [S1, unofficial][S2].

## Operating characteristics

*   **Time Complexity:** The initial sorting and buffer construction is time-consuming [S1, unofficial][S2]. However, during interactive transfer function changes, the cost is reduced to pointer collection and merging, which is fast [S1, unofficial][S2].
*   **Memory Complexity:** Requires storage for index buffers in graphics hardware (video or AGP memory) and pointers in main memory [S1, unofficial][S2].
*   **Measured Performance:** On the UNC Head dataset (2,955,242 voxels), list-based pre-classification achieves 2.80 fps vs 0.3 fps for traditional pre-classification during interactive transfer function changes [S1, unofficial][S2].
*   **Strengths:** Enables interactive rates for transfer function changes by avoiding CPU-GPU bandwidth bottlenecks associated with rebuilding and reloading index buffers [S1, unofficial][S2].
*   **Limitations:** Slightly slower than traditional pre-classification when the transfer function is unchanged [S1, unofficial][S2].
*   **When to Use:** When interactive transfer function changes are required [S1, unofficial][S2].
*   **When Not to Use:** When the transfer function is static, as traditional pre-classification may be slightly faster [S1, unofficial][S2].

## Background

List-based pre-classification was introduced in the context of hardware-accelerated adaptive EWA volume splatting [S1, unofficial][S2]. The basic idea follows the list-based splatting algorithm proposed by Mueller and colleagues [16] [S1, unofficial][S2]. It addresses the limitation of retained-mode hardware pre-classification, where collecting indices and loading the index buffer to graphics hardware is time-consuming and involves significant data transfer between CPU and GPU, preventing interactive visualization of transfer function changes [S1, unofficial][S2].

## Sources

[S1] Hardwareaccelerated adaptive ewa volume splatting, https://core.ac.uk/works/16783031, fetched 2026-10-28 (unofficial)
[S2] Hardware-Accelerated Adaptive EWA Volume Splatting, https://publications.ri.cmu.edu/storage/publications/pub_files/pub4/chen_wei_2004_1/chen_wei_2004_1.pdf, fetched 2026-10-27
