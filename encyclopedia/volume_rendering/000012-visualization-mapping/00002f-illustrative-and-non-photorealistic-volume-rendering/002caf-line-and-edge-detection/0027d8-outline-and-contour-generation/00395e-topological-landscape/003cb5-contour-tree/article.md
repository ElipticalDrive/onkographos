# Contour Tree
Also called: Join Tree, Merge Tree, Split Tree

## Overview
The contour tree is a topological data structure that captures the evolution of connected components in the level sets of a scalar field as the isovalue varies [S1, unofficial]. It solves the problem of representing the global connectivity of scalar data by tracking where contours are created, split, and merged, providing a compact representation of the field's topology [S1, unofficial]. It sets itself apart from the [Reeb Graph](9315-reeb-graph.md) by requiring a simply connected domain and ignoring topological genus (loops), and from the [Joint Contour Net](9316-joint-contour-net.md) by operating on single-variable scalar fields rather than multivariate data [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar field $f$ defined on a domain (typically a grid or simplicial mesh). The domain must be simply connected and free of loops [S1, unofficial].
*   **Output:** A tree structure consisting of vertices (supernodes) and edges (superarcs) [S1, unofficial].

**Data Structures:**
The contour tree is a graph $T = (V, E)$ where:
*   **Vertices ($V$):**
    *   **Leaf Nodes:** Represent local extrema.
        *   Local maxima: Where a contour is created [S1, unofficial].
        *   Local minima: Where a contour is destroyed [S1, unofficial].
    *   **Interior Nodes:** Represent saddle points (critical points of index 1 in 3D).
        *   Type A: A contour splits into two or more disjoint contours (creation of components) [S1, unofficial].
        *   Type B: Two or more disjoint contours merge into one (destruction of components) [S1, unofficial].
*   **Edges ($E$):**
    *   Represent a single continuous contour segment between the supernode where it is created and the supernode where it is destroyed [S1, unofficial].
    *   There is a one-to-one correspondence between individual contours in the data and edges in the tree [S1, unofficial].

**Construction Algorithm (Join Tree Variant):**
A Join Tree captures connected sub-level sets (regions below a specific isovalue) [S1, unofficial]. The construction follows these steps [S1, unofficial]:

1.  **Identify Critical Points:** Find all local maxima and saddle points in the scalar field. Local maxima are identified as critical points where the function value is higher than all neighbors [S1, unofficial].
2.  **Sort Critical Vertices:** Sort the critical vertices by their scalar values in descending order [S1, unofficial]. Optimization: Only sort critical vertices, not all data elements, to improve processing time [S1, unofficial].
3.  **Initialize Tree:** Create leaf nodes for each local maximum. Initialize a disjoint-set data structure (or similar connectivity tracker) for the domain elements [G].
4.  **Sweep and Merge:** Iterate through the sorted critical vertices. For each vertex, update the connectivity of the domain elements associated with it.
    *   If a saddle point causes two previously disconnected regions to become connected as the isovalue decreases, create a new interior node (merge node) in the tree and link the corresponding branches from the two regions to this new node [S1, unofficial].
5.  **Finalize:** Continue until the global minimum is reached. The global minimum is represented by a root node (or final merge node) in the graph [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_join_tree(scalar_field, grid):
    """
    Computes the Join Tree for a scalar field.
    scalar_field: 2D/3D array of scalar values.
    grid: Structure defining connectivity (neighbors) of cells/vertices.
    Returns: Tree structure with nodes and edges.
    """
    # 1. Identify critical points
    local_maxima = find_local_maxima(scalar_field)
    saddle_points = find_saddle_points(scalar_field, grid)
    
    # Combine and sort by value descending
    critical_points = local_maxima + saddle_points
    critical_points.sort(key=lambda p: p.value, reverse=True)
    
    # 2. Initialize data structures
    # Each element in the grid starts in its own component
    components = DisjointSet(grid.num_elements)
    tree_nodes = []
    node_map = {} # Maps component root to tree node index
    
    # Create leaf nodes for local maxima
    for max_pt in local_maxima:
        node = create_tree_node(type='leaf', value=max_pt.value, position=max_pt.pos)
        tree_nodes.append(node)
        # Associate the component containing the maximum with this node
        comp_id = components.find(max_pt.element_id)
        node_map[comp_id] = node.index
        
    # 3. Process saddle points (merges)
    for saddle_pt in saddle_points:
        # Identify the components connected by this saddle
        neighbor_components = set()
        for neighbor in grid.get_neighbors(saddle_pt.element_id):
            neighbor_components.add(components.find(neighbor))
            
        # Remove the component of the saddle itself if it was separate
        saddle_comp = components.find(saddle_pt.element_id)
        neighbor_components.discard(saddle_comp)
        
        # If multiple components merge, create a new interior node
        if len(neighbor_components) > 1:
            new_node = create_tree_node(type='saddle_merge', value=saddle_pt.value, position=saddle_pt.pos)
            tree_nodes.append(new_node)
            
            # Link existing branches to the new node
            for comp_id in neighbor_components:
                if comp_id in node_map:
                    parent_node = node_map[comp_id]
                    add_edge(tree_nodes, parent_node.index, new_node.index)
                else:
                    # Component not yet mapped to a tree node (e.g., pure saddle connection)
                    pass # [G] Handle unconnected saddle logic if necessary
                    
            # Union the components in the disjoint set
            first_comp = neighbor_components.pop()
            for comp_id in neighbor_components:
                components.union(first_comp, comp_id)
                
            # Map the new unified component to the new tree node
            unified_comp = components.find(first_comp)
            node_map[unified_comp] = new_node.index
            
    # 4. Handle global minimum (root)
    # The final remaining component corresponds to the global minimum
    final_comp = components.find(0) # Assuming 0 is a valid element index
    if final_comp in node_map:
        # Link to root if not already done
        pass # [G] Finalize root connection
    else:
        root_node = create_tree_node(type='leaf_min', value=global_min, position=global_min_pos)
        tree_nodes.append(root_node)
        node_map[final_comp] = root_node.index
        
    return tree_nodes
```

**Parameters:**
*   **Neighborhood Definition:** The method of defining neighbors (e.g., 6-connected vs 26-connected in 3D grids) affects the location of critical points but not the underlying structure (number of supernodes and superarcs) [S1, unofficial].
*   **Seeds:** Vertices where contours can be built from. Using seeds reduces the number of seeds required for representation significantly compared to other methods, but often increases the overall storage size of the contour tree [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Genus/Loops:** The contour tree does not typically contain information regarding changes in topological genus (loops). This information must be added to the output separately [S1, unofficial].
*   **Domain Constraints:** The input domain must be simply connected and free of loops. The merge tree computation phase cannot correctly handle data without a boundary or with loops [S1, unofficial].
*   **High Dimensions:** Critical vertices are difficult to identify in data with dimensionality of four or more [S1, unofficial].
*   **Precision:** Changing the method of defining neighbours can slightly alter the location of critical points, though the tree structure remains fixed [S1, unofficial].

## Subtopics

### [Reeb Graph](9315-reeb-graph.md)
The Reeb graph is a topological data structure that generalizes the contour tree to handle scalar fields defined on domains that are not simply connected (i.e., domains containing loops or lacking a boundary). While the contour tree tracks the evolution of connected components of level sets, the Reeb graph tracks the evolution of connected components of level sets modulo homotopy, effectively identifying points in the domain that can be connected by a path of constant scalar value. It solves the problem of capturing the global topology of scalar data in complex geometries, such as toroidal volumes or data with internal voids, where standard contour trees fail or produce ambiguous results. It sets itself apart from the [Contour Tree](9312-contour-tree.md) by explicitly handling genus (loops) in the input domain, and from the [Joint Contour Net](9316-joint-contour-net.md) by operating on single-variable scalar fields rather than multivariate data.

### [Topological Zone Transfer Function Application](11654-topological-zone-transfer-function-application.md)
This method applies distinct transfer functions to different topological zones of a scalar field, determined by mapping sample points to arcs in a contour tree via monotone paths. It sets itself apart from global transfer functions by allowing topology-aware color and opacity mapping, and from simple gradient reversal or flattening by using dynamic tube sweeping to broaden connections between components, providing a smoother approximation of the scalar field.

### Contour-Tree-Defined Segmentation
A segmentation method where each arc of the contour tree represents the union of all contours mapping to it, defining a 'topological zone' as the volume swept out by the contour as its isovalue varies from creation to destruction [S2].

**Steps:**
1.  Compute the contour tree of the underlying trilinear interpolant using a variation of the method from Pascucci and Cole-McLaughlin, applied to the union of cellwise join and split trees [S2].
2.  Construct a widget in each cube where mesh edges are augmented by edges between body saddles, face saddles, and vertices [S2].
3.  Simplify the contour tree to remove symbolic perturbation [S2].
4.  Assign each topological zone a transfer function [S2].
5.  Store the hierarchy of the simplified contour tree using branch decomposition [S2].
6.  Apply trilinear interpolant over hexahedral cells to find the correct opacity in cells spanning multiple topological zones [S2].

**Comparison:**
Unlike Takeshima et al., this method allows independent transfer functions for individual topological zones, uses trilinear interpolation over hexahedral cells (avoiding tetrahedral artifacts), and looks up opacities directly in the transfer function rather than interpolating over tetrahedra [S2].

## Operating characteristics
*   **Time Complexity:** Construction time is dominated by sorting critical vertices. Chiang et al. improve processing time by only sorting critical vertices in the merge tree construction stage, rather than sorting all elements [S1, unofficial].
*   **Memory Complexity:** Using seeds reduces the number of seeds required for representation significantly compared to other methods, but often increases the overall storage size of the contour tree [S1, unofficial].
*   **Performance:** Gueunet et al. present a multithreaded approach ("contour forest" algorithm) for computing the augmented contour tree. Initial evaluations showed quicker computations, but the speedup is limited by load imbalance and redundant computations [S1, unofficial].
*   **Strengths:** Captures topological information as arcs in a tree-like structure, used to compute zero-dimensional persistence measures [S1, unofficial]. Provides a compact representation for feature extraction [S1, unofficial].
*   **Limitations:** Cannot handle domains with loops or without boundaries [S1, unofficial]. Critical point identification is difficult in 4D or higher [S1, unofficial].

## Background
The contour tree was introduced by Boyell and Ruston [S1, unofficial]. It uses concepts from Morse theory to capture changes in the topology of a scalar field at critical points [S1, unofficial]. Merge trees are fundamental data structures in computational topology, often used as a computation step in the construction of more complex topological data structures such as the contour tree [S1, unofficial].

## Sources
[S1] Chapter Topological Visualisation Techniques for Volume Multifield Data, https://archive.org/details/oapen-20.500.12657-49310, fetched 2026-10-27 (unofficial)
[S2] Topology-Controlled Volume Rendering, https://doi.org/10.1109/tvcg.2007.47, fetched 2026-10-27
