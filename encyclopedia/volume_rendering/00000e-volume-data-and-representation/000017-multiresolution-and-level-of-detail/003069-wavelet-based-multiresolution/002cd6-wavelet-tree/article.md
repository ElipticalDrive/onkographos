# Wavelet Tree

Also called: Wavelet-based Multiresolution Hierarchy, Wavelet Volume Tree

## Overview
The Wavelet Tree is a multiresolution data representation that converts a large 3D volume into a hierarchical structure using wavelet transforms to enable level-of-detail (LOD) selection and rendering [S1]. It solves the problem of managing large volumetric datasets by allowing efficient traversal and rendering of selected data blocks at various resolutions [S1]. Unlike standard subdivision hierarchies which may produce topological inaccuracies, this method ensures correct topological information and smaller root-mean-square errors by reorganizing data into wavelet coefficients without requiring additional memory for the hierarchy structure itself [S1]. It builds on [Multiresolution and level of detail](7-multiresolution-and-level-of-detail.md).

## How it works
**Inputs:** A 3D volume dataset; a block size parameter $n$ (number of voxels per leaf block); a user-specified threshold $\tau$ for coefficient pruning.
**Outputs:** A hierarchical tree structure where leaf nodes contain raw volume data, internal nodes contain low-resolution data blocks and wavelet coefficients, and the root contains a single block representing the entire volume.

**Data Structures:**
The tree is a complete 8-ary tree (each node has 8 children).
*   **Leaf Nodes:** Store raw volume blocks of size $n$ voxels.
*   **Internal Nodes:** Store two components:
    1.  A low-resolution data block of size $n$ voxels (derived from the low-pass subbands of its children).
    2.  Wavelet coefficients of size $7n/8$ (derived from the high-pass subbands of its children).
*   **Root Node:** Stores a single low-resolution data block of size $n$ voxels.

**Algorithm Steps:**
1.  **Leaf Initialization:** Subdivide the original 3D volume into a sequence of non-overlapping blocks of equal size, where each block contains $n$ voxels. These raw volume blocks form the leaf nodes of the wavelet tree [S1].
2.  **Leaf Transform:** Perform a 3D wavelet transform on each leaf block. This decomposition produces:
    *   A low-pass filtered subblock of size $n/8$ voxels.
    *   Wavelet coefficients of size $7n/8$ voxels [S1].
3.  **Parent Block Construction:** Group the low-pass filtered subblocks from eight adjacent leaf nodes (siblings in the tree) into a single contiguous block of $n$ voxels. This block becomes the low-resolution data stored in the parent node [S1].
4.  **Coefficient Aggregation:** Collect the wavelet coefficients from the eight child nodes. These coefficients are stored in the parent node alongside the low-resolution data.
5.  **Thresholding:** For each wavelet coefficient in every node, if the absolute value is smaller than the user-specified threshold $\tau$, set the coefficient to zero. This reduces the storage size and complexity of the tree [S1].
6.  **Compression:** Compress the thresholded wavelet coefficients using run-length encoding (RLE) combined with a fixed Huffman encoder [S1].
7.  **Recursive Upward Propagation:** Treat the newly formed low-resolution blocks (from Step 3) as the input data for the next level up. Recursively apply Steps 2–6 in a bottom-up manner until the root is reached. The root node contains a single block of size $n$ representing the entire volume [S1].

**Hierarchical Error Metric:**
To support LOD selection, a hierarchical error metric is calculated for each node [S1].
*   Let $E_p$ be the error of a parent node and $E_{c_i}$ be the error of its $i$-th child ($i=1..8$).
*   The error is calculated as the Mean Square Error (MSE) between the data in the parent node and the data in its eight immediate child nodes.
*   The error value for a node is defined as:
    $$ E_{node} = \text{MSE}(Data_{parent}, Data_{children}) + \max_{i=1..8}(E_{c_i}) $$
    *(Note: The source states the calculation "considers the MSE... adding the maximum error value of the child nodes" [S1]. This implies a recursive accumulation where the parent's error metric includes the worst-case error from its subtree.)*
*   This metric guarantees that $E_{parent} \ge E_{child}$ for all children, ensuring monotonic error bounds up the tree [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_wavelet_tree(volume, block_size_n, threshold_tau):
    """
    Constructs a wavelet tree from a 3D volume.
    volume: 3D array of voxel values.
    block_size_n: Number of voxels per leaf block.
    threshold_tau: Threshold for coefficient pruning.
    Returns: Root node of the wavelet tree.
    """
    # Step 1: Subdivide volume into leaf blocks
    leaf_blocks = subdivide_volume(volume, block_size_n)
    
    # Initialize leaf nodes
    leaf_nodes = []
    for block in leaf_blocks:
        node = Node()
        node.data = block  # Raw volume data
        node.children = []
        node.coefficients = []
        node.error = 0.0
        leaf_nodes.append(node)
        
    # Step 2-6: Bottom-up construction
    current_level_nodes = leaf_nodes
    while len(current_level_nodes) > 1:
        next_level_nodes = []
        # Process nodes in groups of 8
        for i in range(0, len(current_level_nodes), 8):
            children = current_level_nodes[i : i+8]
            if len(children) < 8:
                raise ValueError("Volume dimensions must allow complete 8-way grouping")
            
            parent = Node()
            
            # Step 2: Transform children and collect low-pass subblocks
            low_pass_subblocks = []
            all_coefficients = []
            
            for child in children:
                # Perform 3D wavelet transform on child's data
                low_pass, coeffs = wavelet_transform_3d(child.data)
                low_pass_subblocks.append(low_pass)
                all_coefficients.extend(coeffs)
            
            # Step 3: Group low-pass subblocks into parent data block
            parent.data = combine_subblocks(low_pass_subblocks, block_size_n)
            
            # Step 4 & 5: Store and threshold coefficients
            parent.coefficients = []
            for c in all_coefficients:
                if abs(c) > threshold_tau:
                    parent.coefficients.append(c)
                # else: set to zero (implicit in compression)
            
            # Step 6: Compress coefficients (conceptual step for storage)
            parent.compressed_coeffs = compress(parent.coefficients)
            
            # Step 7: Recursive error calculation
            # Calculate MSE between parent data and reconstructed child data
            # Note: Exact reconstruction requires inverse transform, 
            # but source defines error based on MSE between parent and children data directly.
            mse_val = compute_mse(parent.data, [child.data for child in children])
            
            # Add max error of children
            max_child_error = max(child.error for child in children)
            parent.error = mse_val + max_child_error
            
            # Link parent to children
            parent.children = children
            for child in children:
                child.parent = parent
                
            next_level_nodes.append(parent)
            
        current_level_nodes = next_level_nodes
        
    return current_level_nodes[0]

def wavelet_transform_3d(data_block):
    """
    Performs a 3D wavelet transform on a block of n voxels.
    Returns: (low_pass_subblock, high_pass_coefficients)
    """
    # Implementation depends on specific wavelet basis (e.g., Haar, Daubechies)
    # Standard 3D separable wavelet transform
    # 1. Transform along X
    # 2. Transform along Y
    # 3. Transform along Z
    # Output: 1/8th of data is low-pass (LL L), 7/8th is high-pass (combinations of H)
    pass

def combine_subblocks(subblocks, n):
    """
    Combines 8 low-pass subblocks (each size n/8) into a single block of size n.
    """
    # Interleave or concatenate subblocks to reconstruct the low-resolution volume block
    pass

def compute_mse(data_parent, data_children):
    """
    Computes Mean Square Error between parent data and children data.
    """
    # Align dimensions and compute MSE
    pass

def compress(coefficients):
    """
    Compresses coefficients using RLE and Huffman coding.
    """
    pass
```

## Subtopics

### [Hierarchical Wavelet Tree Construction](6598-hierarchical-wavelet-tree-construction.md)
A bottom-up blockwise wavelet transform and compression scheme used to construct a multiresolution wavelet tree for 3D volume data. It solves the problem of managing large volumetric datasets by enabling parallel rendering of selected data blocks at various levels of detail. Unlike standard subdivision hierarchies which may produce topological inaccuracies, this method ensures correct topological information and smaller root-mean-square errors by reorganizing data into wavelet coefficients without requiring additional memory for the hierarchy structure itself. It builds on [Wavelet Tree](5390-wavelet-tree.md) by providing the specific algorithmic steps for generating the tree structure and coefficients.

### [Wavelet Coefficient Trees](7647-wavelet-coefficient-trees.md)
Wavelet Coefficient Trees provide a hierarchical data structure for volumetric data that exploits the spatial correlation of wavelet coefficients to enable progressive compression and level-of-detail rendering. Unlike standard octrees that split space uniformly, this method defines parent-child relationships based on the geometric refinement of the underlying mesh (tier-based structure) or subband decay (zero-tree structure). It sets itself apart by allowing efficient encoding of insignificant coefficients as "zero trees," where a single symbol represents an entire hierarchy of small coefficients, significantly reducing bitstream size for smooth regions.

### [Huffman-shaped Wavelet Matrix](10172-huffman-shaped-wavelet-matrix.md)
The Huffman-shaped Wavelet Matrix is a compressed index data structure used to provide random access to a stream of node operations in segmentation volume bricks. It maps symbols from a small alphabet to bit patterns using bit-wise negated Canonical Huffman Codes (CHC), stored in a level-wise concatenated bit vector acting as a wavelet matrix. It enables variable bit-length compression and random access, solving the limitation of the parent CSGV method which uses ANS stream compression and requires serial decoding. It sets itself apart from general wavelet matrices by exploiting the specific structure of the CHC to eliminate branching and interval tracking during access queries.

## Operating characteristics
*   **Time Complexity:** Construction is $O(N \log N)$ where $N$ is the total number of voxels, due to the recursive wavelet transforms at each level. Rendering traversal is $O(1)$ per node visited, with early culling based on the hierarchical error metric.
*   **Memory Complexity:** The tree structure itself requires no additional memory overhead for pointers if the tree is implicit or stored in a flat array, as the source notes it does not require "additional memory for the hierarchy structure itself" [S1]. Storage is dominated by the wavelet coefficients, which are compressed.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Guarantees topological correctness; smaller root-mean-square errors compared to subdivision hierarchies; efficient compression via RLE and Huffman; rapid error metric computation for LOD control [S1].
*   **Limitations:** Requires the volume dimensions to be divisible into blocks of size $n$ that allow complete 8-way grouping; thresholding may introduce artifacts if $\tau$ is too high; compression is lossy if coefficients are pruned.
*   **When to use:** Large static or time-varying 3D datasets requiring interactive LOD rendering [S1].
*   **When not to use:** Data with irregular boundaries that do not align with block subdivisions; scenarios requiring exact lossless reconstruction without compression overhead.

## Background
The Wavelet Tree was introduced in the context of hierarchical navigation interfaces for large 3D steady and time-varying data sets [S1]. It originates from the use of wavelet transforms to convert data into hierarchical multiresolution representations [S1]. The method builds upon earlier work on wavelet-based time-space partitioning trees [S1]. The hierarchical error metric used in conjunction with the tree is cited from reference [33] in the source [S1].

## Sources
[S1] Hierarchical Navigation Interface: Leveraging Multiple Coordinated Views, https://academicweb.nd.edu/~cwang11/papers/iv05-hni.pdf, fetched 2026-10-30
