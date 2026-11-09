# Sparse Voxel DAG

Also called: Directed Acyclic Graph (DAG) for voxel encoding, Sparse Voxel Directed Acyclic Graph

## Overview
Sparse Voxel DAG is a compression technique for sparse voxel octrees that reduces memory footprint by merging identical nodes and redirecting parent pointers to single shared instances [S2, unofficial]. It builds on [Sparse Voxel Octree Structure Encoding/Decoding](3012-sparse-voxel-octree-structure-encoding-decoding.md) by introducing node deduplication, distinguishing itself from standard SVOs which store separate nodes for identical subtrees [S2, unofficial].

## How it works
The method extends the Sparse Voxel Octree (SVO) structure by allowing nodes to reference branches of other nodes, effectively creating a Directed Acyclic Graph rather than a strict tree [S1]. This is achieved by identifying identical nodes (subtrees with the same structure and data) and merging them, such that multiple parents point to the same child node [S2, unofficial].

**Inputs and Outputs**
- Input: A Sparse Voxel Octree (SVO) structure.
- Output: A Sparse Voxel DAG structure with reduced node count.

**Data Structures**
The node structure is similar to an SVO node but includes a reference mechanism for identical siblings [S1].
- **Node Fields**:
  - `type`: Enum indicating node type (e.g., empty, internal, leaf) [G].
  - `children`: Array of 8 pointers/references to child nodes [G].
  - `data`: Voxel data (e.g., density, albedo) for leaf nodes [G].
  - `reference_offset`: An integer value used to reference identical nodes on the same level. If 0, the node is unique or points to a standard child. If non-zero, it indicates a shift to an identical sibling [S1].

**Numbered Steps**
1. **Traverse and Identify**: Perform a post-order traversal of the SVO to identify all nodes [G].
2. **Hash/Compare**: For each node, compute a hash or compare its structure and data against previously seen nodes [G].
3. **Merge Identical Nodes**: If a node is identical to an existing node, replace the pointer in the parent with a reference to the existing node [S2, unofficial].
4. **Encode References**: For nodes that are identical to siblings, store a reference offset instead of a full pointer or child data. The reference value indicates the distance in nodes to the referenced node on the same level [S1].
5. **Store Reference Value**: The reference value is determined by the number of zero bits in the parent node's child array between the referencing node and the referenced node. For example, if there are three zero bits between the current node and the target node, the reference value is +4 [S1].

**Formulas**
- Reference Offset Calculation:
  $$ \text{offset} = 2^k $$
  where $k$ is the number of zero bits in the parent node's child array between the referencing node and the referenced node [S1].

**Parameters**
- **Reference Value**: An integer indicating the distance to the identical node. Not given in the sources.

**Edge Cases and Pitfalls**
- **Complex Data**: Merging nodes is less effective when storing complex data like diffuse albedo compared to simple occlusion values, as identical child nodes become rare [S2, unofficial].
- **Build Time**: The process of identifying and merging identical nodes may increase build time [S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_sparse_voxel_dag(svo_root):
    """
    Converts a Sparse Voxel Octree to a Sparse Voxel DAG by merging identical nodes.
    """
    unique_nodes = {}
    
    class Node:
        def __init__(self, node_type, data, children):
            self.type = node_type
            self.data = data
            self.children = children

    def process_node(node):
        if node is None:
            return None
        
        processed_children = []
        for child in node.children:
            processed_children.append(process_node(child))
        
        node_key = (node.type, node.data, tuple(processed_children))
        
        if node_key in unique_nodes:
            return unique_nodes[node_key]
        else:
            new_node = Node(node.type, node.data, processed_children)
            unique_nodes[node_key] = new_node
            return new_node

    return process_node(svo_root)

def traverse_dag_ray(root, origin, direction, cell_size=1.0):
    """
    Traverses a Sparse Voxel DAG along a ray, resolving sibling reference offsets.
    """
    x = int(origin[0] / cell_size)
    y = int(origin[1] / cell_size)
    z = int(origin[2] / cell_size)
    
    step_x = 1 if direction[0] > 0 else -1
    step_y = 1 if direction[1] > 0 else -1
    step_z = 1 if direction[2] > 0 else -1
    
    t_x = (x + (1 if step_x > 0 else 0)) * cell_size - origin[0]
    t_y = (y + (1 if step_y > 0 else 0)) * cell_size - origin[1]
    t_z = (z + (1 if step_z > 0 else 0)) * cell_size - origin[2]
    
    delta_x = abs(cell_size / direction[0]) if direction[0] != 0 else float('inf')
    delta_y = abs(cell_size / direction[1]) if direction[1] != 0 else float('inf')
    delta_z = abs(cell_size / direction[2]) if direction[2] != 0 else float('inf')
    
    while True:
        node = get_node_at(root, x, y, z)
        if node is None or node.type == 'leaf':
            return (x, y, z)
        
        child_idx = ((x & 4) >> 2) | ((y & 4) >> 1) | (z & 4)
        child = node.children[child_idx]
        
        if isinstance(child, int) and child > 0:
            sibling_idx = (child_idx + child) % 8
            child = node.children[sibling_idx]
        
        if child is None or child.type == 'leaf':
            return (x, y, z)
        
        if t_x < t_y:
            if t_x < t_z:
                x += step_x
                t_x += delta_x
            else:
                z += step_z
                t_z += delta_z
        else:
            if t_y < t_z:
                y += step_y
                t_y += delta_y
            else:
                z += step_z
                t_z += delta_z

def get_node_at(root, x, y, z):
    """Helper to resolve DAG references during traversal."""
    node = root
    for depth in range(7, -1, -1):
        if node is None:
            return None
        idx = ((x >> depth) & 1) << 2 | ((y >> depth) & 1) << 1 | (z >> depth) & 1
        child = node.children[idx]
        if isinstance(child, int) and child > 0:
            sibling_idx = (idx + child) % 8
            child = node.children[sibling_idx]
        node = child
    return node
```

## Subtopics

### [Tolerance-based Voxel Fusion](5866-tolerance-based-voxel-fusion.md)
Merges voxels based on tolerance, useful for reducing noise but less effective for structural deduplication.

### [Temporal Difference Encoding](5869-temporal-difference-encoding.md)
Encodes differences between frames, useful for dynamic scenes but not for spatial deduplication.

## Operating characteristics
- **Memory Footprint**: Significantly reduced compared to SVO when there are many identical subtrees [S2, unofficial].
- **Performance Trade-off**: Increased build time due to the need to traverse and compare nodes [S2, unofficial].
- **Utility**: Most effective for simple data like occlusion values; less effective for complex data like diffuse albedo [S2, unofficial].

## Background
The Sparse Voxel DAG was introduced as an extension to the Sparse Voxel Octree (SVO) technique to further decrease memory footprint by merging identical nodes [S2, unofficial].

## Sources
[S1] Voxel Compression - GitHub Pages, https://eisenwave.github.io/voxel-compression-docs/dag/dag.html, fetched 2026-10-27
[S2] Voxel Cone Traced Global Illumination - Leif Node, https://leifnode.com/2015/05/voxel-cone-traced-global-illumination/, fetched 2026-10-27 (unofficial)
