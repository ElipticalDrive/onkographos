# Contour Tree

Also called: Join Tree, Merge Tree, Split Tree

## Overview
The contour tree is a topological data structure that captures the changes in the topology of a scalar field's level sets (contours) as the isovalue varies, specifically tracking how connected components split and merge [S1, unofficial]. It builds on [Topological Landscape](8481-topological-landscape.md) by providing the underlying graph structure where valleys represent contour splits and peaks represent contour merges. Unlike the [Reeb Graph](9315-reeb-graph.md), which generalizes this concept to handle domains with loops or handles, the contour tree is restricted to simply connected domains without boundaries or topological holes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar field defined on a simply connected domain (e.g., a 3D grid or simplicial mesh) [S1, unofficial].
*   **Output:** A tree structure consisting of vertices (supernodes) and edges (superarcs) [S1, unofficial].

**Data Structures:**
The contour tree is a graph $T = (V, E)$ where:
*   **Vertices ($V$):**
    *   **Leaf Nodes:** Represent local extrema.
        *   Local maxima: Where a contour is created.
        *   Local minima: Where a contour is destroyed.
    *   **Interior Nodes:** Represent saddle points (critical points).
        *   Type 1 (Split): A saddle point where one contour splits into two or more disjoint contours (creating new components).
        *   Type 2 (Merge): A saddle point where two or more disjoint contours merge into one (destroying components).
*   **Edges ($E$):**
    *   **Superarcs:** Represent a single connected contour segment between a supernode where it is created and a supernode where it is destroyed.
    *   There is a one-to-one correspondence between individual contours in the data and edges in the tree [S1, unofficial].

**Construction Steps (Join Tree Variant):**
A Join Tree captures connected sub-level sets (regions below a specific isovalue). The construction follows a sweep from global maxima downwards [S1, unofficial]:

1.  **Initialize:** Set the isovalue $v$ to the global maximum of the scalar field.
2.  **Identify Maxima:** Locate all local maxima in the data. Mark these as red critical points (leaf nodes in the tree).
3.  **Sweep Down:** Gradually decrease the isovalue $v$.
4.  **Detect Merges:** As $v$ decreases, monitor the connected components of the sub-level set $\{x | f(x) \le v\}$.
    *   When two distinct regions merge, identify the saddle point where this occurs.
    *   Create an interior node in the tree representing this merge event.
    *   Connect the nodes representing the merging components to this new merge node.
5.  **Repeat:** Continue decreasing $v$ and detecting merges until the global minimum is reached.
6.  **Finalize:** The global minimum is represented by a blue graph node (leaf node). All paths converge here.

**Optimizations and Variants:**
*   **Sorting:** Chiang et al. improve processing time by sorting only the critical vertices during the merge tree construction stage, rather than sorting all data elements [S1, unofficial].
*   **Augmented Contour Tree:** Carr et al. introduced augmented contour trees, which add non-branching vertices at non-critical points to provide additional values for isosurface seeding [S1, unofficial].
*   **Seeds:** To efficiently compute specific contours, "seeds" are identified. A seed is a vertex from which an entire contour for a specified isovalue can be traced, bounded by two supernodes representing the critical points of that contour. Using seeds significantly reduces the number of seeds required for representation but may increase overall storage size [S1, unofficial].
*   **Parallelization:** Gueunet et al. present a multithreaded "contour forest" algorithm for computing augmented contour trees using shared memory. This approach offers quicker computations but is limited by load imbalance and redundant computations [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_join_tree(scalar_field, critical_points):
    """
    Computes a Join Tree for a scalar field.
    
    Args:
        scalar_field: 3D array or mesh of scalar values.
        critical_points: List of tuples (point_index, value, type) where type is 
                         'max', 'min', or 'saddle'.
    
    Returns:
        tree: A graph structure (adjacency list or similar) representing the Join Tree.
    """
    # Initialize tree with leaf nodes for local maxima
    # Sort critical points by value in descending order (global max to min)
    sorted_criticals = sort(critical_points, key=lambda p: p.value, reverse=True)
    
    # Map each critical point to a node in the tree
    node_map = {}
    for cp in sorted_criticals:
        if cp.type == 'max':
            node_map[cp.index] = create_leaf_node('max', cp.value)
        elif cp.type == 'min':
            node_map[cp.index] = create_leaf_node('min', cp.value)
        else:
            node_map[cp.index] = create_interior_node('saddle', cp.value)
            
    # Track active components during the sweep
    # Each component is identified by a representative node
    active_components = {} 
    
    for cp in sorted_criticals:
        current_val = cp.value
        
        if cp.type == 'max':
            # New component starts at local maximum
            comp_id = generate_new_component_id()
            active_components[comp_id] = cp.index
            # Add edge from max node to itself or handle as root of component
            # In Join Tree, max is a leaf. The edge connects to the merge node below.
            
        elif cp.type == 'saddle':
            # Determine which components merge at this saddle
            # This requires analyzing the connectivity of the sub-level set at this value
            merging_components = find_merging_components(scalar_field, cp.index, current_val)
            
            if len(merging_components) > 1:
                # Create a new merge node
                merge_node = node_map[cp.index]
                
                # Connect all merging components to this new merge node
                for comp_id in merging_components:
                    root_node = active_components[comp_id]
                    add_edge(tree, root_node, merge_node)
                    # Update the component representative to the merge node
                    active_components[comp_id] = cp.index
                    
                # If the saddle splits a component (in a general merge tree context),
                # handle accordingly. For Join Tree, we primarily track merges of sub-level sets.
            else:
                # Saddle does not merge distinct components, just modifies existing one
                # Update the component representative
                comp_id = list(merging_components)[0]
                active_components[comp_id] = cp.index
                
        elif cp.type == 'min':
            # Global minimum or local minimum where a component ends
            # In Join Tree, min is a leaf.
            pass

    # Connect remaining active components to the global minimum if not already done
    # The global minimum is typically the root of the Join Tree
    global_min_node = find_global_min_node(critical_points)
    for comp_id, root_node in active_components.items():
        if root_node != global_min_node.index:
            add_edge(tree, root_node, global_min_node.index)
            
    return tree
```

**Parameters:**
*   **Neighborhood Definition:** The method used to define neighbors during split/merge tree computations can slightly alter the geometric location of critical points, though the underlying structure (number of supernodes and superarcs) remains fixed for a given input domain subdivision [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Non-Simply Connected Domains:** The contour tree cannot correctly handle data without a boundary or with loops (topological genus changes) [S1, unofficial]. For such data, the [Reeb Graph](9315-reeb-graph.md) must be used.
*   **High Dimensionality:** Critical vertices are difficult to identify in data with dimensionality of four or more [S1, unofficial].
*   **Genus Information:** The standard contour tree does not contain information regarding changes in topological genus; this must be added to the output separately if required [S1, unofficial].

## Subtopics

### [Reeb Graph](9315-reeb-graph.md)
Generalizes the contour tree to handle domains with loops or handles. Use when the input domain is not simply connected.

### Join Tree
A specific merge tree capturing connected sub-level sets (below isovalue). Use when analyzing features that grow as the isovalue decreases.

### Split Tree
A specific merge tree capturing connected super-level sets (above isovalue). Use when analyzing features that grow as the isovalue increases.

## Operating characteristics
*   **Time Complexity:** Depends on the construction algorithm. Chiang et al.'s optimization of sorting only critical vertices improves processing time for large datasets [S1, unofficial]. Gueunet et al.'s multithreaded "contour forest" algorithm offers speedups but is limited by load imbalance and redundant computations [S1, unofficial].
*   **Memory Complexity:** Using seeds reduces the number of seeds required for representation but often increases the overall storage size of the contour tree compared to other methods [S1, unofficial].
*   **Strengths:** Provides a compact representation of topological changes; useful for feature extraction and transfer function design.
*   **Limitations:** Restricted to simply connected domains; difficult to compute in high dimensions (4D+); sensitive to neighborhood definition for critical point location.
*   **When to Use:** When analyzing scalar fields on simply connected domains (e.g., standard 3D volumes) and needing to track component merging/splitting.
*   **When Not to Use:** When the domain has holes, handles, or no boundary (use [Reeb Graph](9315-reeb-graph.md) instead); when high-dimensional (4D+) data requires precise critical point identification.

## Background
The contour tree was introduced by Boyell and Ruston [S1, unofficial] and uses concepts from Morse theory to capture changes in the topology of scalar fields at critical points [S1, unofficial]. Merge trees, which form the basis for contour trees, are fundamental data structures in computational topology used to compute zero-dimensional persistence measures [S1, unofficial]. The concept has been extended to augmented contour trees by Carr et al. and parallelized by Gueunet et al. [S1, unofficial].

## Sources
[S1] Chapter Topological Visualisation Techniques for Volume Multifield Data, https://archive.org/details/oapen-20.500.12657-49310, fetched 2026-10-27 (unofficial)
