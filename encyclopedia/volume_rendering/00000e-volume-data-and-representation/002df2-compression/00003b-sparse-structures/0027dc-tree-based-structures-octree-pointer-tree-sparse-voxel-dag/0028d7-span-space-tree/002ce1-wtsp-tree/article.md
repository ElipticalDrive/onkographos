# WTSP Tree

Also called: Spatio-Temporal Wavelet Tree Construction, WTSP Tree Construction, WTSP Tree Hierarchical Structure, Wavelet Tree Spatial and Temporal, Wavelet-Based Time-Space Partitioning (WTSP) Tree, Wavelet-Based Time-Space Partitioning (WTSP) Tree Construction

## Overview
The Wavelet-Based Time-Space Partitioning (WTSP) tree is a hierarchical data structure for organizing multiresolution time-varying volumetric data, exploiting both spatial and temporal coherence [S1, S2]. It solves the problem of efficiently storing and retrieving large-scale dynamic volumes by combining a spatial octree with a temporal binary tree [S1, S2]. Unlike methods that process space and time separately, the WTSP tree integrates them into a single unified structure where each spatial node contains a temporal hierarchy spanning the entire time sequence [S1, S2]. It builds on [Span Space Tree](4379-span-space-tree.md) by introducing explicit temporal wavelet decomposition [S1, S2].

## How it works
**Inputs:** Time-varying volumetric data consisting of $T$ time steps. Each time step is a 3D grid of voxels.
**Outputs:** A WTSP tree structure and compressed bitstreams for each node.

**Data Structures:**
The WTSP tree is an octree (spatial hierarchy) of binary trees (temporal hierarchy) [S1, S2].
1.  **Octree Skeleton:** There is only one octree skeleton shared across all time steps [S1, S2].
2.  **Spatial Node Layout:**
    *   **Parent Node:** Stores a low-pass filtered subblock of size $n$ voxels [S1].
    *   **Child Nodes:** Store wavelet coefficients of size $7n/8$ [S1].
    *   The root of the spatial octree represents the entire volume with a single block of size $n$ [S1].
3.  **Temporal Node Layout:**
    *   Attached to each spatial node is a binary time tree [S1, S2].
    *   The time tree spans the entire time sequence $[0, T-1]$ [S1, S2].
    *   Leaf nodes of the time tree correspond to single time steps [S1].
    *   Internal nodes of the time tree store approximation (average) and detail (difference) coefficients from the 1D Haar wavelet transform [S1].
    *   For the root nodes of the spatial octree, the temporal hierarchy is built from low-resolution data blocks rather than wavelet coefficients [S1].

**Parameters:**
*   $n$: Size of the initial data block/subvolume in voxels per dimension [S1].
*   Threshold: User-specified threshold for clamping wavelet coefficients to zero during spatial compression [S1].
*   Haar Wavelets: Used for the 1D temporal transform [S1, S2]. Higher-order wavelets may be used for the 3D spatial transform for better quality, but Haar is standard for temporal [S1].
*   Spatial Error Tolerance ($\epsilon_s$) and Temporal Error Tolerance ($\epsilon_t$): User-specified values at run time to select data blocks with various spatio-temporal resolutions [S2].

**Algorithm Steps:**

1.  **Data Loading:** Load volume data considering one voxel overlapping boundaries between neighboring blocks in each dimension to ensure correct rendering results [S2].
2.  **Stage 1: Spatial Hierarchy Construction (Per Time Step):**
    For each time step $t \in [0, T-1]$:
    1.  Subdivide the volumetric data for time step $t$ into data blocks/subvolumes of size $n$ voxels [S1].
    2.  Recursively apply the following process to build the spatial octree:
        a.  **Uniform Check:** If the current data block is uniform, skip the 3D wavelet transform. Set the low-pass filtered subblock to the uniform value and all corresponding wavelet coefficients to zero [S1].
        b.  **3D Wavelet Transform:** Perform a 3D wavelet transform on the data block. This produces a low-pass filtered subblock of size $n/8$ and wavelet coefficients of size $7n/8$ [S1].
        c.  **Thresholding:** Compare wavelet coefficients against the user-specified threshold. Clamp coefficients smaller than the threshold to zero [S1].
        d.  **Parent Aggregation:** Collect low-pass filtered subblocks from eight neighboring subvolumes (children). Group them into a single block of $n$ voxels to form the parent node [S1].
        e.  **Recursion:** Repeat steps 2a-2d until the root of the octree is reached, where a single block of size $n$ represents the entire volume [S1].
3.  **Stage 2: Temporal Hierarchy Construction (Across Time Steps):**
    1.  Identify octree nodes that have the same spatial location and resolution across all time steps [S1, S2].
    2.  **Non-Root Nodes:** For these nodes, apply 1D wavelet transforms (using Haar wavelets) to the wavelet coefficients associated with the node across the time sequence [S1, S2].
        a.  **Zero Check:** If the wavelet coefficients of the corresponding octree nodes in the time sequence are all zero, skip the 1D wavelet transform process [S1].
        b.  **Compression:** Compress the resulting 1D wavelet transform coefficients using run-length encoding combined with a fixed Huffman encoder [S1].
    3.  **Root Nodes:** For the root nodes of the octrees, build the temporal hierarchy from the low-resolution data blocks rather than wavelet coefficients [S1].
    4.  This process merges all spatial octrees across time into a single unified spatio-temporal hierarchical data structure [S1, S2].
4.  **Error Metric Calculation:**
    During the bottom-up construction, calculate the hierarchical spatial error ($s_e$) and temporal error ($t_e$) for each time tree node based on Mean Square Error (MSE) [S1, S2].
    For a time tree leaf node $T$ (corresponding to a single time step), the spatial error is calculated as:
    $$ s_e(T) = \sum_{i=0}^{7} \text{MSE}(T, T_i) + \max_{i=0}^{7} \{ s_e(T_i) \} $$
    where $T_i$ are the eight spatial child nodes of $T$ in the next spatial level of the octree [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_wtsp_tree(volume_data, block_size_n, threshold):
    """
    volume_data: List of 3D arrays, one for each time step t in [0, T-1]
    block_size_n: Integer, size of the initial block per dimension
    threshold: Float, threshold for clamping wavelet coefficients
    Returns: WTSP tree structure
    """
    T = len(volume_data)
    wtsp_tree = OctreeSkeleton()
    
    # Stage 1: Build Spatial Octree for each time step
    spatial_octrees = []
    for t in range(T):
        octree = build_spatial_octree(volume_data[t], block_size_n, threshold)
        spatial_octrees.append(octree)
        
    # Stage 2: Merge into WTSP Tree
    # Traverse the spatial octree structure (which is identical for all t)
    # Note: In practice, we align nodes by spatial location and resolution
    
    def build_temporal_hierarchy(spatial_node_list):
        """
        spatial_node_list: List of nodes from the same spatial location/resolution
                           across all time steps [octree_t, octree_t+1, ...]
        """
        if not spatial_node_list:
            return None
            
        # Check for uniform/zero optimization
        all_coeffs_zero = all(is_zero_coeffs(node) for node in spatial_node_list)
        if all_coeffs_zero:
            # Skip 1D wavelet transform, store zeros or minimal representation
            return create_zero_temporal_node()
            
        # Extract coefficients or data blocks
        coeffs_or_blocks = [node.get_data_for_temporal_transform() for node in spatial_node_list]
        
        # Apply 1D Haar Wavelet Transform
        # For root nodes, this operates on low-res data blocks
        # For other nodes, it operates on wavelet coefficients
        approx, detail = haar_1d_transform(coeffs_or_blocks)
        
        # Compress detail coefficients
        compressed_detail = compress_huffman_rle(detail)
        
        # Create temporal node
        temp_node = TemporalNode(approx=approx, detail=compressed_detail)
        
        # Recursively build children if not leaf
        if len(spatial_node_list) > 1:
            # Split into left and right children for binary tree
            mid = len(spatial_node_list) // 2
            left_children = build_temporal_hierarchy(spatial_node_list[:mid])
            right_children = build_temporal_hierarchy(spatial_node_list[mid:])
            
            # Note: The binary tree structure is implicit in the Haar transform
            # The 'approx' becomes the parent for the next level up
            # The 'detail' is stored at the current level
            # This logic assumes a bottom-up or top-down construction matching
            # the error tree algorithm [MVW98]
            pass 
            
        return temp_node

    # Helper to align nodes across time steps
    # This requires a canonical ordering of octree nodes
    aligned_nodes = align_spatial_nodes(spatial_octrees)
    
    for node_group in aligned_nodes:
        # node_group is a list of spatial nodes at same location/res across time
        temporal_tree = build_temporal_hierarchy(node_group)
        assign_temporal_tree(node_group[0], temporal_tree)
        
    return wtsp_tree

def build_spatial_octree(volume, block_size_n, threshold):
    """
    Recursively builds the spatial octree for a single time step.
    """
    # Base case: if block is uniform or leaf
    if is_uniform(volume):
        return SpatialNode(low_pass=volume[0], coeffs=[], is_leaf=True)
        
    # Perform 3D Wavelet Transform
    low_pass, coeffs = wavelet_3d_transform(volume)
    
    # Thresholding
    coeffs = [c if abs(c) > threshold else 0 for c in coeffs]
    
    # If leaf node (block_size_n is minimal)
    if block_size_n <= 1:
        return SpatialNode(low_pass=low_pass, coeffs=coeffs, is_leaf=True)
        
    # Recursive step: subdivide
    subvolumes = split_into_8_subvolumes(low_pass, block_size_n // 2)
    children = [build_spatial_octree(sv, block_size_n // 2, threshold) for sv in subvolumes]
    
    # Parent node stores the aggregated low-pass block of size n
    # (This is reconstructed from children's low_pass in the aggregation step)
    # However, the source says parent stores low-pass filtered subblock of size n
    # and children store coeffs.
    # The recursion above builds children first.
    # We need to aggregate low_pass from children to form parent's low_pass?
    # Source: "low-pass filtered subblocks from eight neighboring subvolumes are collected
    # and grouped into a single block of n voxels, which becomes the parent node"
    
    parent_low_pass = aggregate_low_pass(children)
    
    return SpatialNode(low_pass=parent_low_pass, coeffs=coeffs, children=children)

def is_zero_coeffs(node):
    return all(c == 0 for c in node.coeffs)

def is_uniform(volume):
    return len(set(volume.flatten())) == 1

def wavelet_3d_transform(block):
    # Returns low_pass (size n/8) and coeffs (size 7n/8)
    pass

def split_into_8_subvolumes(low_pass, new_size):
    # Split the low_pass block into 8 subvolumes
    pass

def aggregate_low_pass(children):
    # Collect low_pass from 8 children and group into n voxels
    pass

def haar_1d_transform(data_list):
    # Returns approx (average) and detail (difference)
    # For a list of length L, produces L/2 approx and L/2 detail
    # This is recursive for the binary tree
    if len(data_list) == 1:
        return data_list[0], []
    
    approx = [(data_list[i] + data_list[i+1]) / 2 for i in range(0, len(data_list), 2)]
    detail = [(data_list[i] - data_list[i+1]) / 2 for i in range(0, len(data_list), 2)]
    
    # Recursively transform approx to build the tree
    # The detail is stored at the current node
    # The approx is passed up
    next_approx, next_detail = haar_1d_transform(approx)
    
    return next_approx, detail + next_detail

def compress_huffman_rle(data):
    # Run-length encode then Huffman encode
    pass
```

## Subtopics

### [Time-Space Partitioning Tree](5382-time-space-partitioning-tree.md)
The Time-Space Partitioning Tree (TSP) is a hierarchical data structure designed for the compression and rendering of time-varying volumetric data. It addresses the challenge of exploiting both spatial coherence (via an octree) and temporal coherence (via a binary tree) within a single unified structure. Unlike methods that process space and time separately, TSP integrates them by embedding temporal information directly within spatial nodes, allowing for efficient traversal of dynamic volumes. It is the direct predecessor to the WTSP tree.

### [WTSP Tree Traversal and Parallel Rendering](7583-wtsp-tree-traversal-and-parallel-rendering.md)
This method traverses the Wavelet-based Time-Space Partitioning (WTSP) tree at run time to reconstruct and render data blocks of varying spatio-temporal resolutions in parallel, composing the final image. It solves the problem of efficiently visualizing large-scale time-varying data on parallel clusters by eliminating parent-child data dependencies for reconstruction and balancing workload among processors. It is distinguished from its parent [WTSP Tree](5371-wtsp-tree.md) by its runtime execution strategy and from its sibling [Time-Space Partitioning Tree](5382-time-space-partitioning-tree.md) by its specific use of wavelet compression and error-guided bucketization for distributed rendering.

### Hierarchical Spatial and Temporal Error Metric
Spatial error ($s_e$) and temporal error ($t_e$) are calculated for each time tree node during the bottom-up construction of the WTSP tree [S1]. The metric is based on Mean Square Error (MSE) [S1]. For a time tree leaf node $T$, the spatial error is the sum of the MSE between the data at node $T$ and the data of the same time step at $T$'s eight spatial child nodes in the next spatial level of the octree, plus the maximum spatial error of the child nodes [S1]. Users specify separate spatial and temporal error tolerances to select data blocks with various spatio-temporal resolutions [S2].

## Operating characteristics
**Time Complexity:** The construction involves a 3D wavelet transform for each time step and each spatial block, followed by a 1D wavelet transform across time for each spatial node. The complexity is proportional to the total number of voxels times the number of time steps, $O(N \cdot T \cdot \log N)$ for spatial and $O(N \cdot \log T)$ for temporal per node [S1, S2].
**Memory Complexity:** The WTSP tree stores low-pass blocks at parent nodes and wavelet coefficients at children. Compression via run-length encoding and Huffman coding reduces the storage of wavelet coefficients, especially if many are zeroed out by the threshold [S1].
**Performance:** The method exhibits good compression ratios if many consecutive zero subsequences are present in the wavelet coefficient sequence, and is very fast to decompress [S1].
**Strengths:** Exploits both spatial and temporal coherence; allows flexible spatio-temporal level-of-detail selection [S1, S2].
**Limitations:** Trade-off between image quality and reconstruction cost when choosing higher-order wavelets for the spatial domain [S1].

## Background
The WTSP tree originates from the Time-Space Partitioning (TSP) tree [SCM99] [S1, S2]. It was introduced by Wang, Shen, et al. in 2004/2005 for large-scale time-varying data visualization [S1, S2]. The method builds on the wavelet tree concept [GWGS02] and the error tree algorithm [MVW98].

## Sources
[S1] A Multiresolution Volume Rendering Framework for Large- ..., https://diglib.eg.org/bitstream/handle/10.2312/VG.VG05.011-019/011-019.pdf?sequence=1, fetched 2026-10-26
[S2] PDF Hierarchical Navigation Interface: Leveraging Multiple Coordinated ..., https://academicweb.nd.edu/~cwang11/papers/iv05-hni.pdf, fetched 2026-10-30
