# Wavelet Tree

Also called: Wavelet-based Multiresolution Tree, Hierarchical Wavelet Tree

## Overview
A wavelet tree is a multiresolution data representation that converts a large 3D volume into a hierarchical structure using wavelet transforms to enable level-of-detail (LOD) selection and efficient rendering [S1]. It solves the problem of managing large volumetric datasets by allowing the renderer to traverse the tree and select data blocks at various levels of detail, balancing image quality and rendering speed [S1]. Unlike standard subdivision hierarchies which may produce topological inaccuracies, this method ensures correct topological information and smaller root-mean-square errors by reorganizing data into wavelet coefficients without requiring additional memory for the hierarchy structure itself [S1]. It builds on [Wavelet-based Multiresolution](6266-wavelet-based-multiresolution.md) by providing the specific algorithmic steps for generating the tree structure and coefficients.

## How it works
**Inputs:** A 3D volume dataset; a block size $n$ (number of voxels per leaf block); a user-specified threshold $T$ for coefficient pruning.
**Outputs:** A binary tree structure where leaf nodes contain raw volume data, internal nodes contain low-resolution approximation data and compressed wavelet coefficients, and a root node containing the coarsest approximation.

**Data Structures:**
The tree is a binary tree (conceptually, though the construction groups 8 children).
*   **Leaf Node:** Stores a raw volume block of size $n$ voxels.
*   **Internal Node:**
    *   `low_res_data`: A volume block of size $n$ voxels (the low-pass filtered approximation of its 8 children).
    *   `coefficients`: A list of wavelet coefficients of size $7n/8$ from the 8 children, thresholded and compressed.
    *   `error`: A scalar value representing the hierarchical error metric for this node.
*   **Root Node:** Stores a single low-resolution block of size $n$ voxels representing the entire volume.

**Algorithm Steps:**

1.  **Leaf Initialization:** Subdivide the original 3D volume into a sequence of non-overlapping blocks of size $n$ voxels. These raw blocks form the leaf nodes of the wavelet tree [S1].
2.  **Leaf Transform:** Perform a 3D wavelet transform on each leaf block. This produces:
    *   One low-pass filtered subblock of size $n/8$ voxels.
    *   Seven high-pass detail subbands (wavelet coefficients) with a total size of $7n/8$ voxels [S1].
3.  **Parent Aggregation (Bottom-Up):** For every group of eight adjacent leaf nodes (or internal nodes at the current level):
    *   Gather the low-pass filtered subblocks from the eight children.
    *   Group these eight subblocks (each size $n/8$) into a single block of $n$ voxels. This block becomes the `low_res_data` for the parent node [S1].
    *   Gather the wavelet coefficients from the eight children.
4.  **Recursive Transform:** Recursively apply Steps 2 and 3 to the newly created parent nodes. Continue this bottom-up process until a single root node is reached, which contains a low-resolution block of size $n$ representing the entire volume [S1].
5.  **Thresholding:** For every node in the tree (except the root, which has no coefficients to prune), compare each wavelet coefficient magnitude against the user-specified threshold $T$. If $|c| < T$, set $c = 0$. This reduces the storage size of the coefficients [S1].
6.  **Compression:** Compress the thresholded wavelet coefficients in each node using run-length encoding (RLE) combined with a fixed Huffman encoder [S1].
7.  **Error Calculation:** Calculate the hierarchical error metric for each internal node. This metric is used to guide LOD selection during rendering [S1].

**Hierarchical Error Metric Formula:**
For an internal node $P$ with eight immediate child nodes $C_1, \dots, C_8$:
$$ E(P) = \max_{i=1}^{8} \left( \text{MSE}(P.\text{low\_res\_data}, C_i.\text{raw\_data}) \right) $$
Where $\text{MSE}(A, B)$ is the Mean Square Error between the low-resolution data of the parent and the raw data (or reconstructed data) of the child. The sources state that this metric guarantees $E(P) \ge E(C_i)$ for all children, ensuring the error bound is monotonic up the tree [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_wavelet_tree(volume, block_size_n, threshold_T):
    """
    Constructs a wavelet tree from a 3D volume.
    volume: 3D array of voxels.
    block_size_n: Number of voxels per leaf block (e.g., 8^3 = 512).
    threshold_T: Scalar threshold for coefficient pruning.
    """
    # Step 1: Create leaf nodes
    leaves = []
    blocks = chunk_volume(volume, block_size_n)
    for block in blocks:
        leaves.append(Node(type='leaf', raw_data=block))

    # Step 2 & 3: Bottom-up construction
    current_level_nodes = leaves
    while len(current_level_nodes) > 1:
        next_level_nodes = []
        # Process groups of 8
        for i in range(0, len(current_level_nodes), 8):
            children = current_level_nodes[i : i+8]
            if len(children) < 8:
                # Handle edge case: incomplete group at end of volume
                # Not specified in sources; assume padding or skip [G]
                continue
            
            parent = Node(type='internal')
            
            # Step 2: Transform children to get low-pass and coefficients
            low_pass_subblocks = []
            all_coefficients = []
            
            for child in children:
                if child.type == 'leaf':
                    # Perform 3D Wavelet Transform on raw data
                    low_pass, coeffs = wavelet_transform_3d(child.raw_data)
                else:
                    # For internal nodes, the 'raw_data' is effectively the 
                    # reconstructed data from its children, but for the 
                    # parent's low-res aggregation, we use the child's 
                    # stored low_res_data.
                    # Note: The source says "low-pass filtered subblocks from 
                    # eight adjacent leaf nodes... grouped into parent".
                    # Recursively, for internal nodes, we use their low_res_data.
                    low_pass = child.low_res_data
                    # Coefficients are already stored/compressed in child
                    # We need to decompress to aggregate? 
                    # Source says: "Group the low-pass filtered subblocks... 
                    # from eight adjacent leaf nodes". 
                    # For higher levels, it implies using the low-res data 
                    # of the children.
                    coeffs = child.coefficients # Compressed
                
                low_pass_subblocks.append(low_pass)
                all_coefficients.extend(coeffs)
            
            # Step 3: Group low-pass subblocks into parent's low_res_data
            # 8 subblocks of size n/8 make one block of size n
            parent.low_res_data = merge_subblocks(low_pass_subblocks, block_size_n)
            
            # Step 5: Threshold coefficients
            # Decompress if necessary, threshold, then re-compress
            # Assuming coeffs are stored as raw values for this step
            thresholded_coeffs = []
            for c in all_coefficients:
                if abs(c) < threshold_T:
                    thresholded_coeffs.append(0)
                else:
                    thresholded_coeffs.append(c)
            
            # Step 6: Compress
            parent.coefficients = compress(thresholded_coeffs, method='RLE_Huffman')
            
            # Step 7: Calculate Error
            # MSE between parent's low_res_data and each child's raw/reconstructed data
            # For leaf children, raw_data is known. For internal, we use their 
            # low_res_data as the approximation of their subtree? 
            # Source: "MSE between the data in a parent node and the data in 
            # its eight immediate child nodes".
            # This implies comparing parent.low_res_data vs child.low_res_data 
            # (if child is internal) or child.raw_data (if child is leaf).
            # However, the source says "adding the maximum error value of the 
            # child nodes".
            
            max_error = 0
            for child in children:
                if child.type == 'leaf':
                    child_data = child.raw_data
                else:
                    child_data = child.low_res_data # Approximation of child's subtree
                
                mse = mean_square_error(parent.low_res_data, child_data)
                if mse > max_error:
                    max_error = mse
            
            parent.error = max_error
            next_level_nodes.append(parent)
        
        current_level_nodes = next_level_nodes

    return current_level_nodes[0] # Root

def wavelet_transform_3d(data_block):
    """
    Performs a 3D wavelet transform on a block of size n.
    Returns: (low_pass_subblock, detail_coefficients)
    low_pass_subblock size: n/8
    detail_coefficients size: 7n/8
    """
    # Implementation depends on specific wavelet (e.g., Haar, Daubechies)
    # Not specified in sources [G]
    pass

def merge_subblocks(subblocks, n):
    """
    Merges 8 subblocks of size n/8 into a single block of size n.
    """
    # Implementation depends on spatial arrangement
    # Not specified in sources [G]
    pass

def compress(data, method):
    """
    Compresses data using RLE and Huffman coding.
    """
    # RLE encoding
    rle_data = run_length_encode(data)
    # Huffman coding
    huffman_data = huffman_encode(rle_data)
    return huffman_data

def mean_square_error(a, b):
    """
    Computes MSE between two data blocks.
    """
    # a and b must be same size
    sum_sq_diff = 0
    count = 0
    for i in range(len(a)):
        diff = a[i] - b[i]
        sum_sq_diff += diff * diff
        count += 1
    return sum_sq_diff / count
```

**Edge Cases and Pitfalls:**
*   **Volume Dimensions:** The volume must be divisible into blocks of size $n$. If not, padding or boundary handling is required (not specified in sources) [G].
*   **Incomplete Groups:** At the end of a level, there may be fewer than 8 nodes. The source does not specify how to handle this; typically, these nodes are promoted or handled as a special case [G].
*   **Precision:** Floating-point precision in wavelet transforms and MSE calculations can affect error bounds and compression ratios [S1].

## Subtopics

### [Hierarchical Wavelet Tree Construction](6598-hierarchical-wavelet-tree-construction.md)
Hierarchical Wavelet Tree Construction is a bottom-up blockwise wavelet transform and compression scheme used to construct a multiresolution wavelet tree for 3D volume data. It solves the problem of managing large volumetric datasets by enabling parallel rendering of selected data blocks at various levels of detail. Unlike standard subdivision hierarchies which may produce topological inaccuracies, this method ensures correct topological information and smaller root-mean-square errors by reorganizing data into wavelet coefficients without requiring additional memory for the hierarchy structure itself. It builds on [Wavelet Tree](5390-wavelet-tree.md) by providing the specific algorithmic steps for generating the tree structure and coefficients.

### Wavelet Coefficient Trees
Wavelet Coefficient Trees define parent-child relationships for wavelet coefficients with respect to a slow growing refinement structure, rooted in mesh elements of the base (tier 0) mesh [S2]. This variant is used for progressive compression of volumetric subdivision meshes [S2].
*   **Tier 2 Tree Structure:** Tier 2 vertices are associated with edge centers of the tier 0 mesh. A coarse wavelet coefficient at tier 3 has only tier 3 descendants associated with that edge [S2].
*   **Tier 1 Tree Structure:** Tier 1 vertices are associated with face centers of the tier 0 mesh. A tier 2 wavelet at level $l$ has two sets of child coefficients at level $l+1$: one child wavelet for each face produced by dividing the tier 0 face, and one child wavelet for each edge introduced into the tier 0 face [S2].
*   **Tier 0 Tree Structure:** Tier 0 wavelets are associated with tier 0 cell centers. A tier 0 wavelet at level $l$ has three sets of children at level $l+1$: one wavelet for each subcell the tier 0 cell is divided into, one wavelet for each face inserted to produce those subcells, and one wavelet for each internal edge required to subdivide the tier 0 cell [S2].

## Operating characteristics
*   **Time Complexity:** Construction is $O(N \log N)$ where $N$ is the total number of voxels, due to the hierarchical wavelet transform. Rendering traversal is $O(1)$ per node visited, with LOD selection guided by the error metric [S1].
*   **Memory Complexity:** The tree structure itself requires minimal additional memory beyond the stored coefficients and low-res data. Compression (RLE + Huffman) significantly reduces the storage size of the detail coefficients [S1].
*   **Measured Performance:** The sources do not provide specific hardware or resolution benchmarks for the wavelet tree construction itself, but note that the error metric can be rapidly computed [S1].
*   **Strengths:** Enables efficient multiresolution rendering; guarantees topological correctness; error metric is monotonic and easy to compute [S1].
*   **Limitations:** Requires careful handling of volume boundaries and incomplete groups; compression efficiency depends on the sparsity of wavelet coefficients after thresholding [S1].
*   **When to Use:** Large volumetric datasets where LOD rendering is required and topological accuracy is critical [S1].
*   **When Not to Use:** Small datasets where the overhead of tree construction and traversal outweighs the benefits; scenes requiring exact lossless reconstruction without compression artifacts [S1].

## Background
The wavelet tree was introduced as a multiresolution data representation for large 3D steady and time-varying data sets [S1]. It originates from the work on hierarchical navigation interfaces for volume rendering, leveraging wavelet transforms for level-of-detail selection [S1]. The method was developed to address the limitations of standard subdivision hierarchies, which could suffer from topological inaccuracies [S1]. The hierarchical error metric used in conjunction with the wavelet tree was also introduced in this context to control the trade-off between image quality and rendering speed [S1].

## Sources
[S1] Hierarchical Navigation Interface: Leveraging Multiple Coordinated Views, https://academicweb.nd.edu/~cwang11/papers/iv05-hni.pdf, fetched 2026-10-30
[S2] Progressive Compression of Volumetric Subdivision Meshes, https://digital.library.unt.edu/ark:/67531/metadc1405535/?q=volume data compression rendering, fetched 2026-10-28
