# List-based Pre-classification

Also called: Bucket sort pre-classification, Iso-value index buffers, List-based Pre-classification Performance

## Overview

List-based pre-classification is an optimization for interactive transfer function changes in volume rendering that avoids rebuilding index buffers on the CPU by pre-sorting voxel indices into iso-value buckets, allowing fast pointer merging [S1, unofficial][S2]. Unlike naive pre-classification which requires rebuilding and reloading the index buffer on every transfer function change (causing CPU-GPU bandwidth bottlenecks), this method uses pre-sorted buffers and pointer merging for interactive rates [S1, unofficial][S2]. It builds on [Real-time transfer function evaluation](1551-real-time-transfer-function-evaluation.md) by providing a mechanism to efficiently select which pre-classified voxels are visible based on the current transfer function configuration.

## How it works

**Inputs and Outputs:**
*   **Input:** A 3D volume dataset with density values; a traversal order (e.g., front-to-back slices); proxy geometry indices corresponding to voxels.
*   **Output:** Pre-loaded iso-value index buffers in graphics hardware (video or AGP memory) and pointers to the start of each iso-value array in main memory.

**Data Structures:**
1.  **Iso-value Buckets:** The range of possible density/iso-values is divided into 256 intervals [S1, unofficial][S2].
2.  **Index Buffers:** Arrays of vertex indices for proxy geometry (splat vertices). These are stored in graphics hardware (video memory or AGP memory) [S1, unofficial][S2]. The placement in AGP memory does not affect performance [S1, unofficial][S2].
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
    """
    Pre-process volume data for list-based pre-classification.
    
    Args:
        volume_data: 3D array of density values.
        traversal_slices: List of slices in traversal order.
        
    Returns:
        index_buffers: List of arrays, each containing indices for a specific iso-value bucket.
        pointers: Array of pointers to the start of each bucket in index_buffers.
    """
    num_buckets = 256
    # Initialize empty lists for each bucket
    # Each bucket stores indices of proxy geometry vertices
    bucket_lists = [[] for _ in range(num_buckets)]
    
    # Step 1 & 2: Bucket sort and compute iso-value lists per slice
    for slice_data in traversal_slices:
        for voxel_index, density in enumerate(slice_data):
            # Map density to one of 256 intervals
            # Assuming density is normalized to [0, 255] or mapped via transfer function range
            bucket_id = map_density_to_bucket(density)
            bucket_id = clamp(bucket_id, 0, num_buckets - 1)
            
            # Step 3: Rearrange indices into iso-value index buffers
            # voxel_index corresponds to the proxy geometry index for this voxel
            bucket_lists[bucket_id].append(voxel_index)
            
    # Step 4: Pre-load buffers (conceptual, actual implementation depends on GPU API)
    # index_buffers = load_to_gpu(bucket_lists)
    
    # Step 5: Maintain pointers in main memory
    # Pointers point to the start of each iso-value array
    pointers = [0] * num_buckets
    current_offset = 0
    for i in range(num_buckets):
        pointers[i] = current_offset
        current_offset += len(bucket_lists[i])
        
    return bucket_lists, pointers

def update_for_transfer_function(pointers, bucket_lists, visible_buckets):
    """
    Merge visible voxel indices for a new transfer function.
    
    Args:
        pointers: Array of pointers to the start of each bucket.
        bucket_lists: List of index arrays for each bucket.
        visible_buckets: List of bucket IDs that are visible under the new transfer function.
        
    Returns:
        merged_indices: Concatenated list of indices for visible voxels.
    """
    merged_indices = []
    
    # Step 6: Collect appropriate pointers and merge them quickly
    for bucket_id in visible_buckets:
        start_ptr = pointers[bucket_id]
        # Get the indices for this bucket
        # Note: In a real implementation, pointers might point to offsets in a single large buffer
        # or directly to the lists. Here we assume direct access for clarity.
        indices = bucket_lists[bucket_id]
        merged_indices.extend(indices)
        
    return merged_indices

def map_density_to_bucket(density):
    """
    Map a density value to a bucket ID [0, 255].
    Implementation detail: [G]
    """
    # Assuming density is in [0, 1]
    return int(density * 255)

def clamp(value, min_val, max_val):
    return max(min_val, min(value, max_val))
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
*   **Measured Performance:** On the UNC Head dataset (2,955,242 voxels), list-based pre-classification achieves 2.80 fps vs 0.3 fps for traditional pre-classification during interactive transfer function changes [S1, unofficial][S2]. Other datasets show similar trends: Checkerboard (786,432 voxels) 2.86 fps vs 0.12 fps; Lobster (555,976 voxels) 10.60 fps vs 1.19 fps [S1, unofficial][S2].
*   **Strengths:** Enables interactive rates for transfer function changes by avoiding CPU-GPU bandwidth bottlenecks associated with rebuilding and reloading index buffers [S1, unofficial][S2].
*   **Limitations:** Slightly slower than traditional pre-classification when the transfer function is unchanged [S1, unofficial][S2].
*   **When to Use:** When interactive transfer function changes are required.
*   **When Not to Use:** When the transfer function is static, as traditional pre-classification may be slightly faster.

## Background

List-based pre-classification was introduced in the context of hardware-accelerated adaptive EWA volume splatting [S1, unofficial][S2]. The basic idea follows the list-based splatting algorithm proposed by Mueller and colleagues [16] [S1, unofficial][S2]. It addresses the limitation of retained-mode hardware pre-classification, where collecting indices and loading the index buffer to graphics hardware is time-consuming and involves significant data transfer between CPU and GPU, preventing interactive visualization of transfer function changes [S1, unofficial][S2].

## Sources

[S1] Hardwareaccelerated adaptive ewa volume splatting, https://core.ac.uk/works/16783031, fetched 2026-10-28 (unofficial)
[S2] Hardware-Accelerated Adaptive EWA Volume Splatting, https://publications.ri.cmu.edu/storage/publications/pub_files/pub4/chen_wei_2004_1/chen_wei_2004_1.pdf, fetched 2026-10-27
