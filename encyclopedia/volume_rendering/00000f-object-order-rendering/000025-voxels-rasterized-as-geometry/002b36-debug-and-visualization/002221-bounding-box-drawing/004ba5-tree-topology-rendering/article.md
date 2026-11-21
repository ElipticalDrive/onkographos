# Tree Topology Rendering
Also called: Grid Topology Visualization, Node Wireframe Rendering
## Overview
Tree Topology Rendering visualizes the internal hierarchical structure of an OpenVDB grid by rendering wireframe bounding boxes for every node (both leaf and non-leaf) in the tree [S1, unofficial]. It serves as a debugging and visualization tool to inspect the spatial distribution and granularity of the voxel data, distinguishing itself from standard volume rendering by exposing the sparse data structure's geometry rather than the scalar field values [S1, unofficial]. It builds on [Bounding Box Drawing](2733-bounding-box-drawing.md) by automating the generation of these boxes for the entire tree hierarchy.
## How it works
### Inputs and Outputs
**Inputs:**
- An OpenVDB `Grid` object containing the tree structure to visualize [S1, unofficial].
- A transformation matrix mapping index space to world space [S1, unofficial].
- A node color lookup table with 4 entries [S1, unofficial].

**Outputs:**
- GPU buffers containing vertex positions, colors, and indices for rendering the wireframe boxes [S1, unofficial].

### Data Structures
The method generates three buffers for GPU upload [S1, unofficial]:
1.  **Points Buffer:** `std::vector<GLfloat>` of size `nodeCount * 24`. Stores 3 floats per vertex (x, y, z). Since each box has 8 vertices, this implies 3 vertices per box are stored? No, 8 vertices * 3 floats = 24 floats. So `nodeCount * 24` floats total.
2.  **Colors Buffer:** `std::vector<GLfloat>` of size `nodeCount * 24`. Stores 3 floats per vertex (r, g, b).
3.  **Indices Buffer:** `std::vector<GLuint>` of size `nodeCount * 24`. Stores 2 unsigned ints per edge. A cube has 12 edges. 12 edges * 2 vertices/edge = 24 indices. So `nodeCount * 24` indices total.

### Parameters
- **Node Colors (`sNodeColors`):** An array of 4 color entries [S1, unofficial].
    - Level 0 nodes use index 3: `(0.00608299, 0.279541, 0.625)` [S1, unofficial].
    - Level 1 nodes use index 2: `(0.871, 0.394, 0.01916)` [S1, unofficial].
    - Colors for levels 2 and 3 are not specified in the sources [S1, unofficial].

### Steps
1.  Initialize empty vectors for points, colors, and indices [S1, unofficial].
2.  Iterate over all nodes in the grid tree using `cbeginNode` [S1, unofficial].
3.  For each node:
    a.  Get the node's bounding box in index space [S1, unofficial].
    b.  Compute the minimum and maximum world coordinates for the node box by subtracting 0.5 from the index-space minimum and adding 0.5 to the index-space maximum, then converting to world space [S1, unofficial].
    c.  Generate 8 vertex positions for the box corners by converting the 8 index-space corners to world space [S1, unofficial].
    d.  Generate 12 indices for the box edges using the 8 vertex indices [S1, unofficial].
    e.  Assign a color to the node based on its level using the `sNodeColors` lookup table [S1, unofficial].
    f.  Append the 8 vertices and their corresponding colors to the points and colors buffers [S1, unofficial].
    g.  Append the 12 indices to the indices buffer [S1, unofficial].
4.  Upload the generated points, colors, and indices to GPU buffers [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_tree_topology(grid, transform_matrix, s_node_colors):
    """
    Renders the wireframe bounding boxes for all nodes in an OpenVDB grid.
    
    Args:
        grid: OpenVDB Grid object.
        transform_matrix: 4x4 matrix transforming index space to world space.
        s_node_colors: List of 4 RGB tuples for node levels 0-3.
        
    Returns:
        points: List of floats (vertex positions).
        colors: List of floats (vertex colors).
        indices: List of ints (edge indices).
    """
    points = []
    colors = []
    indices = []
    
    # Iterate over all nodes in the grid tree
    for node in grid.cbeginNode():
        # Get the bounding box of the current node in index space
        bbox = node.bbox()
        
        # Compute min and max world coordinates
        # Subtract 0.5 from index-space min, add 0.5 to index-space max
        min_idx = bbox.min() - 0.5
        max_idx = bbox.max() + 0.5
        
        # Convert to world space
        min_world = transform_matrix * min_idx
        max_world = transform_matrix * max_idx
        
        # Generate 8 vertex positions for the box corners in world space
        # Corners are combinations of min_world and max_world for x, y, z
        corners = [
            (min_world.x, min_world.y, min_world.z),
            (max_world.x, min_world.y, min_world.z),
            (min_world.x, max_world.y, min_world.z),
            (max_world.x, max_world.y, min_world.z),
            (min_world.x, min_world.y, max_world.z),
            (max_world.x, min_world.y, max_world.z),
            (min_world.x, max_world.y, max_world.z),
            (max_world.x, max_world.y, max_world.z)
        ]
        
        # Assign color based on node level
        level = node.level()
        if level < len(s_node_colors):
            color = s_node_colors[level]
        else:
            color = (0.0, 0.0, 0.0) # Default color if level out of range [G]
            
        # Add vertices and colors to buffers
        for corner in corners:
            points.extend(corner)
            colors.extend(color)
            
        # Generate 12 indices for the box edges
        # Edges connect: 0-1, 1-3, 3-2, 2-0 (bottom), 4-5, 5-7, 7-6, 6-4 (top), 0-4, 1-5, 3-7, 2-6 (vertical)
        edge_indices = [
            0, 1, 1, 3, 3, 2, 2, 0, # Bottom face
            4, 5, 5, 7, 7, 6, 6, 4, # Top face
            0, 4, 1, 5, 3, 7, 2, 6  # Vertical edges
        ]
        
        # Add indices to buffer
        indices.extend(edge_indices)
        
    return points, colors, indices
```

### Edge Cases and Pitfalls
- **Floating Point Equality:** Floating point equality warnings may occur during ground plane rendering and should be suppressed [S1, unofficial].
- **Level Out of Range:** If a node's level exceeds the number of entries in `sNodeColors`, a default color should be used [G].
- **Empty Grid:** If the grid has no nodes, the buffers will be empty, and no geometry will be rendered [G].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of nodes in the grid, $O(N)$, where $N$ is the total number of nodes [S1, unofficial].
- **Memory Complexity:** Linear with respect to the number of nodes, $O(N)$, for storing the vertex, color, and index buffers [S1, unofficial].
- **Performance:** Performance depends on the number of nodes in the grid. Sparse grids with few nodes will render quickly, while dense grids with many nodes may cause performance issues due to the large number of draw calls or buffer sizes [G].
- **Strengths:** Provides immediate visual feedback on the tree structure and node distribution [S1, unofficial].
- **Limitations:** Only visualizes the structure, not the data values [S1, unofficial]. May become cluttered for very dense grids [G].
- **When to Use:** Debugging and visualizing the internal structure of OpenVDB grids [S1, unofficial].
- **When Not to Use:** When the goal is to visualize the scalar field values or when performance is critical for very large grids [G].

## Background
Tree Topology Rendering is a technique used in the OpenVDB library for debugging and visualization purposes [S1, unofficial]. It was developed as part of the OpenVDB project to help users understand the spatial distribution of data within the grid [S1, unofficial].

## Sources
[S1] AcademySoftwareFoundation/openvdb, https://github.com/AcademySoftwareFoundation/openvdb, fetched 2026-10-27 (unofficial)
