# Wavelet Tree

Also called: Wavelet-based Multiresolution Tree, Hierarchical Wavelet Tree

## Overview
A Wavelet Tree is a multiresolution data representation that converts a large 3D volume into a hierarchical structure using wavelet transforms to enable level-of-detail (LOD) selection and efficient rendering [S1]. It differs from sibling methods like [Projecting Detail Coefficients](1802-projecting-detail-coefficients.md) by explicitly organizing data into a tree where internal nodes store aggregated low-resolution approximations and compressed detail coefficients, rather than projecting coefficients directly to the screen. It builds on the principles of [Wavelet-based Multiresolution](6266-wavelet-based-multiresolution.md) by providing a concrete tree topology for hierarchical navigation.

## How it works
**Inputs:** A 3D volume dataset.
**Outputs:** A tree structure where leaf nodes contain raw volume blocks, internal nodes contain low-resolution data blocks and compressed wavelet coefficients, and the root contains a single block representing the entire volume.

**Data Structures:**
*   **Node:** Contains a data block (low-resolution approximation) and a list of wavelet coefficients.
*   **Tree:** A hierarchical structure where each internal node has exactly eight children (in a 3D octree-like subdivision).
*   **Leaf Nodes:** Store raw volume blocks of size $n$ voxels.
*   **Internal Nodes:** Store a low-resolution data block of size $n$ voxels and wavelet coefficients of size $7n/8$ (per child block).
*   **Root Node:** Stores a single block of size $n$ voxels.

**Parameters:**
*   $n$: The number of voxels in a leaf block.
*   $T$: User-specified threshold for wavelet coefficients. Coefficients with magnitude $|c| < T$ are set to zero [S1].

**Algorithm Steps:**
1.  **Subdivision:** Subdivide the original 3D volume into a sequence of blocks of the same size, assuming each has $n$ voxels. These raw volume blocks form the leaf nodes of the wavelet tree [S1].
2.  **Leaf Transform:** Perform a 3D wavelet transform on each leaf block. This produces a low-pass filtered subblock of size $n/8$ and wavelet coefficients of size $7n/8$ [S1].
3.  **Parent Aggregation:** Group the low-pass filtered subblocks from eight adjacent leaf nodes into a single block of $n$ voxels. This aggregated block becomes the low-resolution data block stored in the parent node [S1].
4.  **Recursive Construction:** Recursively apply the 3D wavelet transform and subblock grouping process in a bottom-up manner until the root is reached. The root node contains a single block of size $n$ representing the entire volume [S1].
5.  **Thresholding:** For every node in the tree, set wavelet coefficients to zero if their magnitude is smaller than the user-specified threshold $T$ [S1].
6.  **Compression:** Compress the thresholded wavelet coefficients in each tree node using run-length encoding combined with a fixed Huffman encoder [S1].
7.  **Error Calculation:** Calculate a hierarchical error metric for each node to guide LOD selection. The error for a parent node is defined as the maximum of the error values of its eight immediate child nodes, where the error of a child node is the Mean Square Error (MSE) between the parent's low-resolution data and the child's raw data [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_wavelet_tree(volume, block_size_n, threshold_T):
    """
    Constructs a Wavelet Tree from a 3D volume.
    
    Args:
        volume: 3D array of voxel values.
        block_size_n: Number of voxels in a leaf block.
        threshold_T: Threshold for coefficient pruning.
        
    Returns:
        root_node: The root of the constructed Wavelet Tree.
    """
    # Step 1: Subdivide volume into leaf blocks
    leaf_blocks = subdivide_volume(volume, block_size_n)
    
    # Create leaf nodes
    leaf_nodes = []
    for block in leaf_blocks:
        node = Node()
        node.data = block  # Raw volume block
        node.children = []
        node.coefficients = []
        node.error = 0.0
        leaf_nodes.append(node)
        
    # Step 2 & 3: Bottom-up construction
    current_level_nodes = leaf_nodes
    while len(current_level_nodes) > 1:
        next_level_nodes = []
        # Process groups of 8 nodes
        for i in range(0, len(current_level_nodes), 8):
            child_nodes = current_level_nodes[i:i+8]
            if len(child_nodes) != 8:
                # Handle edge case: incomplete group at the end
                # Not specified in sources; assume padding or error
                raise ValueError("Volume dimensions must allow complete 8-node grouping")
            
            parent_node = Node()
            
            # Step 2: Transform leaf blocks to get low-pass subblocks
            low_pass_subblocks = []
            for child in child_nodes:
                # Perform 3D wavelet transform
                low_pass, coeffs = wavelet_transform_3d(child.data)
                child.coefficients = coeffs
                low_pass_subblocks.append(low_pass)
            
            # Step 3: Group low-pass subblocks into parent data block
            # The source states: "Group the low-pass filtered subblocks ... into a single block of n voxels"
            # This implies a reconstruction or aggregation step. 
            # Typically, 8 subblocks of size n/8 are combined to form a block of size n.
            parent_node.data = combine_subblocks(low_pass_subblocks)
            
            # Step 7: Calculate Error Metric
            # Error of a child = MSE(parent.data, child.data)
            # Error of parent = max(error of children)
            max_child_error = 0.0
            for child in child_nodes:
                mse = mean_square_error(parent_node.data, child.data)
                child.error = mse
                if mse > max_child_error:
                    max_child_error = mse
            parent_node.error = max_child_error
            
            # Link parent to children
            parent_node.children = child_nodes
            for child in child_nodes:
                child.parent = parent_node
                
            next_level_nodes.append(parent_node)
            
        current_level_nodes = next_level_nodes
        
    root_node = current_level_nodes[0]
    
    # Step 5: Thresholding
    # Apply threshold to all nodes except root (root has no coefficients to prune in this context, 
    # or root coefficients are handled separately if present)
    # The source says "wavelet coefficients in each tree node will be set to zero..."
    # We traverse the tree to apply thresholding
    apply_threshold(root_node, threshold_T)
    
    # Step 6: Compression
    compress_coefficients(root_node)
    
    return root_node

def wavelet_transform_3d(data_block):
    """
    Performs a 3D wavelet transform on a data block.
    Returns low-pass subblock and wavelet coefficients.
    """
    # Implementation depends on specific wavelet basis (e.g., Haar, Daubechies)
    # Not specified in sources. [G]
    # Assuming a standard separable 3D wavelet transform
    low_pass = apply_low_pass_filter_3d(data_block)
    coeffs = apply_high_pass_filters_3d(data_block)
    return low_pass, coeffs

def combine_subblocks(subblocks):
    """
    Combines 8 low-pass subblocks of size n/8 into a single block of size n.
    """
    # Implementation depends on the specific aggregation method.
    # The source implies a direct grouping. [G]
    # Assuming simple concatenation or reconstruction based on the transform inverse
    # without the high-pass components.
    return reconstruct_from_low_pass(subblocks)

def mean_square_error(data1, data2):
    """
    Calculates the Mean Square Error between two data blocks.
    """
    n = len(data1)
    mse = sum((v1 - v2)**2 for v1, v2 in zip(data1, data2)) / n
    return mse

def apply_threshold(node, threshold):
    """
    Sets wavelet coefficients to zero if their magnitude is below threshold.
    """
    if node.coefficients:
        node.coefficients = [c if abs(c) >= threshold else 0 for c in node.coefficients]
    for child in node.children:
        apply_threshold(child, threshold)

def compress_coefficients(node):
    """
    Compresses wavelet coefficients using run-length encoding and Huffman coding.
    """
    if node.coefficients:
        # Run-length encoding
        rle_data = run_length_encode(node.coefficients)
        # Huffman coding
        node.compressed_coefficients = huffman_encode(rle_data)
    for child in node.children:
        compress_coefficients(child)
```

## Subtopics

### [Hierarchical Wavelet Tree Construction](6598-hierarchical-wavelet-tree-construction.md)
A bottom-up blockwise wavelet transform and compression scheme used to construct a multiresolution wavelet tree for volume data, allowing for parallel rendering of selected data blocks at various levels of detail.

## Operating characteristics
*   **Time Complexity:** Construction is $O(N \log N)$ where $N$ is the total number of voxels, due to the hierarchical wavelet transform. Rendering traversal is proportional to the number of nodes visited, which depends on the LOD selection strategy and error threshold.
*   **Memory Complexity:** Storage is reduced by thresholding and compression. The uncompressed tree requires $O(N)$ space. Compressed storage depends on the sparsity of coefficients after thresholding.
*   **Strengths:** Enables efficient LOD selection and error-bounded rendering. The hierarchical error metric guarantees that the error of a parent node is greater than or equal to its children, facilitating predictable quality control [S1].
*   **Limitations:** Requires careful selection of the threshold $T$ to balance compression and quality. The bottom-up construction may be memory-intensive for very large volumes if not implemented in a streaming or block-wise manner.
*   **When to use:** Large 3D volumes where interactive rendering with LOD is required, and error bounds are important.
*   **When not to use:** Small volumes where the overhead of tree construction and traversal outweighs the benefits.

## Background
The Wavelet Tree was introduced in the context of hierarchical navigation interfaces for large 3D steady and time-varying data sets [S1]. It leverages wavelet transforms to convert data into a hierarchical multiresolution representation, allowing for efficient level-of-detail selection and rendering [S1]. The method is coupled with a hierarchical error metric to control the trade-off between image quality and rendering speed [S1].

## Sources
[S1] Hierarchical Navigation Interface: Leveraging Multiple Coordinated Views, https://academicweb.nd.edu/~cwang11/papers/iv05-hni.pdf, fetched 2026-10-30
