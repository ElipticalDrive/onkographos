# Hierarchical Wavelet Tree Construction

Also called: Bottom-up Blockwise Wavelet Transform and Compression, Bottom-up blockwise wavelet transform, Multiresolution wavelet tree construction, Wavelet Tree Construction

## Overview
Hierarchical Wavelet Tree Construction is a bottom-up blockwise wavelet transform and compression scheme used to construct a multiresolution wavelet tree for volume data [S1, S2]. It solves the problem of storing and rendering large 3D volumes by recursively decomposing the data into blocks, allowing for parallel rendering of selected data blocks at various levels of detail [S1]. It sets itself apart from flat storage or non-hierarchical transforms by organizing the volume into a tree where parent nodes represent lower-resolution approximations of their children, enabling efficient level-of-detail selection [S1, S2].

## How it works
**Inputs:** A 3D volume dataset.
**Outputs:** A hierarchical wavelet tree structure and compressed bit streams for wavelet coefficients.

**Data Structures:**
*   **Volume Block:** A 3D array of voxels. The algorithm assumes each raw volume block has $n$ voxels, where $n = 2^i \times 2^j \times 2^k$ for integers $i, j, k > 0$ [S1, S2].
*   **Tree Node:** Represents a block at a specific resolution. It contains:
    *   A low-pass filtered subblock of size $n/8$ [S1, S2].
    *   Wavelet coefficients of size $7n/8$ [S1, S2].
    *   A compressed bit stream of the thresholded wavelet coefficients [S1, S2].
    *   An error metric value (calculated as Mean Square Error between the low-resolution block and the raw data) [S1, S2].

**Parameters:**
*   **Threshold ($T$):** A user-provided value used to prune insignificant wavelet coefficients. Coefficients with magnitude $|c| < T$ are set to zero [S1, S2].

**Steps:**

1.  **Subdivision:** Subdivide the original 3D data into raw volume blocks of size $n$ voxels. These blocks form the leaf nodes of the wavelet tree [S1, S2].
2.  **Leaf Transform:** Perform a 3D wavelet transform on each leaf block. This produces:
    *   A low-pass filtered subblock of size $n/8$ [S1, S2].
    *   Wavelet coefficients of size $7n/8$ [S1, S2].
3.  **Parent Aggregation:** Collect the low-pass filtered subblocks from eight adjacent leaf nodes in the wavelet tree. Group them into a single block of $n$ voxels. This new block becomes the low-resolution data block associated with the parent node in the next level of the wavelet hierarchy [S1, S2].
4.  **Recursion:** Recursively apply the 3D wavelet transform and subblock grouping process to the parent blocks until the root is reached [S1, S2].
    *   **Root Condition:** At the root, a single block of size $n$ represents the entire volume. Since the root has no parent, no 3D wavelet transform is performed [S1, S2].
5.  **Uniformity Check (Optimization):** Before performing the 3D wavelet transform on any block, check if the data block is uniform [S1, S2].
    *   If uniform, skip the 3D wavelet transform computation [S1, S2].
    *   Set the low-pass filtered subblock to that uniform value [S1, S2].
    *   Set all corresponding wavelet coefficients to zero [S1, S2].
6.  **Thresholding:** Compare the wavelet coefficients associated with a tree node against the user-provided threshold $T$ [S1, S2]. Set coefficients smaller than $T$ to zero to reduce storage size [S1, S2].
7.  **Compression:** Compress the thresholded wavelet coefficients using run-length encoding combined with a fixed Huffman encoder [S1, S2]. Save the compressed bit stream into an individual file or structure [S1, S2].
8.  **Error Calculation:** Calculate an error value at every tree node based on the Mean Square Error (MSE) between the low-resolution data block $B$ and the corresponding raw data subvolume [S1, S2].

**Reconstruction (Run-time):**
To reconstruct a data block at a certain resolution:
1.  Retrieve the low-pass filtered subblock of size $n/8$ from its parent node. This may entail recursive requests of low-resolution data blocks from ancestor nodes [S1, S2].
2.  Obtain the wavelet coefficients of size $7n/8$ by decoding the corresponding bit stream [S1, S2].
3.  Group the low-pass filtered subblock and the wavelet coefficients [S1, S2].
4.  Apply an inverse 3D wavelet transform to reconstruct the data block [S1, S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def construct_wavelet_tree(volume_data, block_size_n, threshold):
    """
    Constructs a hierarchical wavelet tree from 3D volume data.
    
    Args:
        volume_data: 3D array of voxels.
        block_size_n: Number of voxels per block (n = 2^i * 2^j * 2^k).
        threshold: User-provided threshold for coefficient pruning.
        
    Returns:
        root_node: The root node of the constructed wavelet tree.
    """
    # Determine dimensions based on block_size_n
    # Assume volume is divisible into blocks of size n
    # For simplicity, assume n = 8^L for some integer L, so 8 children per parent
    # In general, n = 2^i * 2^j * 2^k. The grouping step assumes 8 adjacent leaves
    # form a parent. This implies the block structure is hierarchical with branching factor 8.
    
    # Step 1: Subdivide into leaf blocks
    leaf_blocks = divide_into_blocks(volume_data, block_size_n)
    
    # Step 2-4: Bottom-up construction
    current_level_nodes = leaf_blocks
    tree_levels = []
    
    while len(current_level_nodes) > 1:
        next_level_nodes = []
        
        # Group 8 adjacent nodes into a parent
        # Assuming a linearized array where 8 consecutive nodes are siblings
        for i in range(0, len(current_level_nodes), 8):
            children = current_level_nodes[i : i+8]
            if len(children) < 8:
                # Handle boundary case: not enough children for a full parent
                # This depends on specific volume dimensions. 
                # Standard implementation assumes power-of-8 dimensions or padding.
                # For this algorithm description, we assume valid input.
                pass 
            
            parent_node = create_parent_node(children, threshold)
            next_level_nodes.append(parent_node)
            
        tree_levels.append(next_level_nodes)
        current_level_nodes = next_level_nodes
        
    root_node = current_level_nodes[0]
    return root_node

def create_parent_node(children, threshold):
    """
    Creates a parent node from 8 child leaf nodes.
    """
    # Step 3: Collect low-pass subblocks from children
    # Each child has a low-pass subblock of size n/8
    # Group them into a single block of size n
    low_pass_parent_block = group_low_pass_subblocks(children)
    
    # Step 5: Uniformity Check on the parent block (before transform)
    # Note: The algorithm description says "check uniformity of the data block"
    # before transform. In the bottom-up pass, we are creating the parent
    # from children's low-pass outputs. The transform happens on the PARENT
    # to create the GRANDPARENT.
    # So, we first compute the parent's representation (low-pass block + coeffs)
    # Then, if this parent is to be used as a child for a higher level,
    # we check its uniformity.
    
    # However, the text says: "After performing a 3D wavelet transform to each block...
    # low-pass subblocks... collected... become parent node... Recursively apply...
    # To save space... check uniformity of the data block."
    # This implies the check happens on the block being transformed.
    
    # Let's refine: The loop processes nodes to create their PARENTS.
    # The "block" being transformed is the parent block formed by grouping.
    # Wait, the text says: "After performing a 3D wavelet transform to each block...
    # low-pass subblocks... collected... become parent node".
    # This means the LEAVES are transformed first. Then their low-pass outputs
    # are grouped to form the PARENT's data block. Then the PARENT's data block
    # is transformed to create the GRANDPARENT.
    
    # So, for the current `children` (leaves or lower-level nodes):
    # 1. They already have their low-pass subblocks and coefficients (from previous step or leaf init).
    # 2. We group their low-pass subblocks to form `low_pass_parent_block`.
    # 3. We need to transform `low_pass_parent_block` to get the parent's coefficients?
    # No, the text says: "The low-pass filtered subblocks from eight adjacent leaf nodes...
    # collected and grouped into a single block of n voxels, which will become the low resolution
    # data block associated with the parent node... We recursively apply this 3D wavelet transform
    # and subblock grouping process".
    
    # This implies:
    # Level 0 (Leaves): Transform raw data -> Low-pass (n/8) + Coeffs (7n/8).
    # Level 1 (Parents): Group 8 Level-0 Low-passes -> Block (n). Transform Block (n) -> Low-pass (n/8) + Coeffs (7n/8).
    # ...
    
    # So, `create_parent_node` takes 8 children.
    # It gathers their low-pass subblocks to form the input block for the parent's transform.
    
    input_block = group_low_pass_subblocks(children)
    
    # Step 5: Check uniformity of input_block
    if is_uniform(input_block):
        uniform_value = input_block[0,0,0]
        parent_low_pass = fill_block(uniform_value, n_over_8)
        parent_coeffs = [0] * (7 * n_over_8) # All zeros
        parent_compressed = compress_coeffs(parent_coeffs, threshold) # Trivial
    else:
        # Step 2 (Recursive): Perform 3D wavelet transform on input_block
        low_pass, coeffs = perform_3d_wavelet_transform(input_block)
        
        # Step 6: Thresholding
        thresholded_coeffs = apply_threshold(coeffs, threshold)
        
        # Step 7: Compression
        parent_compressed = compress_coeffs(thresholded_coeffs, threshold)
        
        parent_low_pass = low_pass
        parent_coeffs = thresholded_coeffs # Stored for reconstruction if needed, but compressed is primary
        
    # Step 8: Error Calculation
    # MSE between low_pass (downsampled) and original raw data of the subvolume
    # This requires access to the original raw data for this region.
    error_metric = calculate_mse(parent_low_pass, original_raw_data_for_region)
    
    return Node(
        low_pass_subblock=parent_low_pass,
        compressed_bitstream=parent_compressed,
        error_metric=error_metric,
        children=children
    )

def reconstruct_block(node):
    """
    Reconstructs the data block for a given node.
    """
    # Step 1: Retrieve low-pass subblock from parent
    # If node is root, low-pass is the block itself (or derived from it)
    # If node is not root, get low-pass from parent
    if node.is_root():
        # Root has no parent. The root block IS the low-res representation.
        # But to reconstruct the full block at the root level, we need the root's coeffs?
        # No, root has no coeffs. The root block is the final approximation.
        # If we want to reconstruct the block *at* the root, it's just the root's data.
        # But usually, we reconstruct a block *associated* with a node.
        # The node stores a low-pass subblock (n/8) and coeffs (7n/8).
        # Inverse transform combines them.
        pass
    
    # Get low-pass from parent
    if not node.is_root():
        low_pass = node.parent.low_pass_subblock
    else:
        # For the root, the "low-pass" is the entire block if we consider the root
        # as the result of the previous level's grouping? 
        # Actually, the root node *is* the block of size n.
        # It doesn't have a parent to get low-pass from.
        # The reconstruction logic in [S1] says: "low-pass filtered subblock... retrieved from its parent".
        # For the root, this step is skipped or the root's block is the result.
        low_pass = node.low_pass_subblock # Or the block itself if n/8 == n (level 0?)
        
    # Step 2: Decode wavelet coefficients
    coeffs = decode_bitstream(node.compressed_bitstream)
    
    # Step 3 & 4: Inverse Transform
    reconstructed_block = inverse_3d_wavelet_transform(low_pass, coeffs)
    
    return reconstructed_block
```

## Subtopics

### [Trilinear B-spline Wavelet Encoding](7390-trilinear-b-spline-wavelet-encoding.md)
A multiresolution data representation technique for n-dimensional data (specifically n=3) that combines d/z subdivision with trilinear B-spline wavelets to improve approximation quality on coarser levels of detail compared to standard subdivision without wavelets.

## Operating characteristics
*   **Time Complexity:** The construction is $O(N)$ where $N$ is the total number of voxels, as each voxel is processed a constant number of times in the hierarchical transform [S1, S2]. The uniformity check provides an optimization for smooth regions [S1, S2].
*   **Memory Complexity:** The storage is reduced by thresholding and compression. The tree structure itself adds overhead for pointers and metadata, but the compressed coefficients significantly reduce the footprint compared to raw data [S1, S2].
*   **Measured Performance:** The bit-level run-length encoding combined with Huffman coding exhibits good compression ratios when many consecutive zero subsequences are present, which is common after thresholding [S1, S2]. Decompression is very fast [S1, S2].
*   **Strengths:** Enables parallel rendering of selected data blocks at various levels of detail [S1]. Efficient storage through hierarchical compression [S1, S2].
*   **Limitations:** Requires volume dimensions to be compatible with the block size $n = 2^i \times 2^j \times 2^k$ and the grouping factor of 8 [S1, S2]. Boundary conditions (non-power-of-8 dimensions) require handling [G].
*   **When to use:** Large 3D volume datasets requiring interactive or parallel rendering with level-of-detail control [S1, S2].
*   **When not to use:** Data that does not compress well with wavelets or requires exact reconstruction without thresholding artifacts [S1, S2].

## Background
The hierarchical wavelet tree construction algorithm is similar to methods described in [9, 26] (referenced in [S1]) and [GWGS02, WS04] (referenced in [S2]) [S1, S2]. It was developed as part of parallel multiresolution volume rendering algorithms for large data sets [S1, S2]. The approach leverages the multiresolution properties of wavelets to allow efficient traversal and rendering of volumetric data [S1, S2].

## Sources
[S1] A Parallel Multiresolution Volume Rendering Algorithm for Large Data ..., https://academicweb.nd.edu/~cwang11/papers/pc05-pmvr.pdf, fetched 2026-10-26
[S2] Parallel Multiresolution Volume Rendering of Large Data ..., https://academicweb.nd.edu/~cwang11/papers/pgv04-pmvr.pdf, fetched 2026-10-26
