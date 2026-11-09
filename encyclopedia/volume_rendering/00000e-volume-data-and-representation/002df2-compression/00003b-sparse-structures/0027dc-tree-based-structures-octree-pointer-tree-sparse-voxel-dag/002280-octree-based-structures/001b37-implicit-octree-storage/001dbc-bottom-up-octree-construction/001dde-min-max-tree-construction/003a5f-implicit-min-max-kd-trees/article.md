# Implicit Min/Max KD-Trees
Also called: Wald et al

## Overview
Implicit Min/Max KD-Trees provide a memory-efficient acceleration structure for volume rendering by mapping a kd-tree directly onto the volume's voxel grid, storing min/max scalar ranges at each node to enable early ray traversal culling. Unlike explicit pointer-based trees, this method eliminates pointer storage overhead by using an implicit array-based layout, making it suitable for large datasets on commodity hardware [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D volume dataset of dimensions $W \times H \times D$.
- Output: An implicit kd-tree structure stored in a linear array, where each node contains min/max scalar values.

**Data Structures:**
The tree is stored in a flat array. For a node at index $i$ (0-based), children are at $2i+1$ and $2i+2$. Each node stores:
- `min_val`: Minimum scalar value in the subtree.
- `max_val`: Maximum scalar value in the subtree.
- `voxel_count`: Number of voxels in the subtree (optional, for normalization).

**Construction Steps:**
1. **Build Base KD-Tree:** Construct a kd-tree over all voxels such that split planes coincide with cell boundaries. This ensures a one-to-one mapping between volume cells and kd-tree leaves [S1, unofficial].
   - Splitting heuristic: At each node, split the volume along the dimension with the largest extent at the center of the current bounding box. Choose the split plane at the cell boundary closest to the center in that dimension [S1, unofficial].
2. **Compute Min/Max Values:** Recursively compute min/max values for all nodes.
   - Leaf nodes: Store the scalar value of the associated voxel.
   - Inner nodes: Store the min/max of their children's min/max values.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_implicit_minmax_kdtree(volume, dims):
    """
    volume: 3D array of scalar values
    dims: tuple (W, H, D)
    Returns: list of nodes, each node is (min_val, max_val)
    """
    # Initialize tree array with None
    # Estimate size: 2 * num_leaves - 1 for a full binary tree
    num_leaves = dims[0] * dims[1] * dims[2]
    tree_size = 2 * num_leaves
    tree = [None] * tree_size
    
    def build_node(node_idx, bounds):
        """
        bounds: (x_min, x_max, y_min, y_max, z_min, z_max)
        """
        if bounds is leaf:
            # Leaf node: store voxel value
            voxel_val = volume[bounds]
            tree[node_idx] = (voxel_val, voxel_val)
            return
        
        # Determine split dimension and plane
        dim = largest_extent_dimension(bounds)
        split_plane = closest_cell_boundary_to_center(bounds, dim)
        
        # Split bounds
        left_bounds, right_bounds = split_bounds(bounds, dim, split_plane)
        
        # Recursively build children
        left_child_idx = 2 * node_idx + 1
        right_child_idx = 2 * node_idx + 2
        
        build_node(left_child_idx, left_bounds)
        build_node(right_child_idx, right_bounds)
        
        # Compute min/max for current node
        left_min, left_max = tree[left_child_idx]
        right_min, right_max = tree[right_child_idx]
        tree[node_idx] = (min(left_min, right_min), max(left_max, right_max))
    
    root_bounds = (0, dims[0], 0, dims[1], 0, dims[2])
    build_node(0, root_bounds)
    return tree
```

**Parameters:**
- Splitting heuristic: Center-based split in the largest dimension [S1, unofficial].
- Typical values: Not specified.

**Edge Cases and Pitfalls:**
- Precision: Floating-point comparisons for min/max should be consistent.
- Boundaries: Ensure split planes align with voxel boundaries to maintain the one-to-one mapping [S1, unofficial].
- Degenerate input: Empty volumes or single-voxel volumes should be handled as base cases.

## Subtopics

### [Implicit KD-Tree for Iso-Surface Ray Tracing](8881-implicit-kd-tree-for-iso-surface-ray-tracing.md)
A memory-efficient acceleration structure for iso-surface ray tracing that uses an implicit balanced binary kd-tree to store min/max values for early subtree culling, avoiding storage of pointers and splitting planes.

## Operating characteristics
**Time and Memory Complexity:**
- Construction: $O(N \log N)$ where $N$ is the number of voxels.
- Memory: $O(N)$ for the implicit tree array, significantly less than pointer-based trees.

**Measured Performance:**
- Single PC (Dual-CPU): Bonsai 3.4 fps, Aneurysm 3.0 fps, ML 512^3 1.2 fps, Female 2.7 fps, Female (zoom) 2.3 fps [S1, unofficial].
- 5-Node Cluster: Bonsai 16.2 fps, Aneurysm 14.6 fps, ML 512^3 6.1 fps, Female 13.6 fps, Female (zoom) 11.2 fps [S1, unofficial].

**Scaling:**
- Scales well with cluster computing, showing significant speedup on multi-node setups [S1, unofficial].

**Strengths and Limitations:**
- Strengths: Low memory footprint, fast construction, suitable for large datasets.
- Limitations: Performance may degrade with highly anisotropic data or complex scenes requiring fine-grained culling.

**When to Use:**
- Use for large volume datasets where memory is a constraint and interactive frame rates are desired.

**When Not to Use:**
- Avoid for scenes with complex geometry or when high precision is required for fine details.

## Background
Originating from Wald et al. [WFM* 05], this method builds on earlier work by Wilhelm and van Gelder [WV92] for surface ray tracing. It was designed to enable interactive volume rendering on commodity hardware by reducing memory overhead [S1, unofficial].

## Sources
[S1] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
