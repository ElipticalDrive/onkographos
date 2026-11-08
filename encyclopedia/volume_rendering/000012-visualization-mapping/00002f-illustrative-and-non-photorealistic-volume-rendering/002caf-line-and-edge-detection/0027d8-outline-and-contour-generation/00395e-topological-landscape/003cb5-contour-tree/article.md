# Contour Tree

Also called: Join Tree, Merge Tree, Split Tree

## Overview
The contour tree is a topological data structure that captures the changes in the topology of a scalar field's level sets (contours) as the isovalue sweeps across the scalar range [S1, unofficial]. It solves the problem of tracking how connected components of these level sets are created, split, and merged, providing a one-to-one correspondence between tree edges and individual contours in the data [S1, unofficial]. It sets itself apart from the [Reeb Graph](9315-reeb-graph.md) by requiring a simply connected domain without loops, and from the [Joint Contour Net](9316-joint-contour-net.md) by operating on single-variable scalar fields rather than multivariate data.

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar field defined on a simply connected domain (e.g., a 3D grid or simplicial mesh). The domain must be free of loops [S1, unofficial].
*   **Output:** A tree structure consisting of vertices (supernodes) and edges (superarcs) [S1, unofficial].

**Data Structures:**
The contour tree is a graph where:
*   **Vertices (Supernodes):**
    *   **Leaf Nodes:** Represent local extrema.
        *   Local maxima: Where a contour is created [S1, unofficial].
        *   Local minima: Where a contour is destroyed [S1, unofficial].
    *   **Interior Nodes:** Represent saddle points where connectivity changes.
        *   Split saddle: One contour splits into two or more disjoint contours [S1, unofficial].
        *   Merge saddle: Two or more disjoint contours merge into one [S1, unofficial].
*   **Edges (Superarcs):** Represent a single contour segment between the supernode where it is created and the supernode where it is destroyed [S1, unofficial].
*   **Seeds (Optional Optimization):** Vertices from which entire contours for a specified isovalue can be computed. A contour is bounded by two supernodes representing the critical points of that contour [S1, unofficial].

**Algorithm Steps:**
The construction typically involves building a merge tree (specifically a Join Tree or Split Tree) and then deriving the contour tree.

1.  **Identify Critical Points:** Locate all local maxima, local minima, and saddle points in the scalar field. Note that identifying critical vertices becomes difficult in data with dimensionality of four or more [S1, unofficial].
2.  **Construct Merge Tree (Join Tree Variant):**
    *   Gradually decrease the isovalue from the global maxima [S1, unofficial].
    *   Identify local maxima as critical points (often visualized as red nodes) [S1, unofficial].
    *   As the isovalue decreases, monitor the connected components of the sub-level sets (regions below the isovalue) [S1, unofficial].
    *   When two distinct regions merge, mark the merge event by creating a node in the graph [S1, unofficial].
    *   Continue until reaching the global minima (visualized as a blue graph node) [S1, unofficial].
    *   *Optimization:* To improve processing time, only sort critical vertices during the merge tree construction stage rather than sorting all elements [S1, unofficial].
3.  **Derive Contour Tree:** The contour tree is derived from the merge tree structure. The leaves of the contour tree correspond to the local extrema identified in step 1. The interior nodes correspond to the saddle points where splits or merges occurred [S1, unofficial].
4.  **Augmentation (Optional):** Non-branching vertices can be added at non-critical points to create an "augmented contour tree," providing additional values for isosurfaces to be seeded from [S1, unofficial].
5.  **Seed Computation (Optional):** If seeds are used, compute them to allow tracing an entire contour for a specified isovalue from a single element bounded by two supernodes [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_contour_tree(scalar_field, domain):
    """
    Computes the contour tree for a scalar field on a simply connected domain.
    Returns a tree structure with supernodes and superarcs.
    """
    # Step 1: Identify critical points
    critical_points = find_critical_points(scalar_field, domain)
    # Note: find_critical_points must handle 3D grid/mesh topology.
    # In 4D+, this step is difficult and may be inaccurate [S1, unofficial].
    
    # Step 2: Build Join Tree (sweeping from max to min)
    # Initialize tree with local maxima as leaves
    join_tree = MergeTree()
    for max_point in critical_points.local_maxima:
        join_tree.add_leaf(max_point)
        
    # Sort critical vertices to optimize construction [S1, unofficial]
    sorted_criticals = sort_by_scalar_value(critical_points.saddles_and_minima, descending=True)
    
    # Step 3: Process merges
    # As isovalue decreases, track connected components of sub-level sets
    # When components merge, add merge node to join_tree
    for saddle_or_min in sorted_criticals:
        components_affected = get_components_affected_by(saddle_or_min, scalar_field)
        if len(components_affected) > 1:
            merge_node = join_tree.create_merge_node(saddle_or_min)
            for comp in components_affected:
                join_tree.connect(merge_node, comp)
        else:
            # Propagate component through saddle
            join_tree.propagate(components_affected[0], saddle_or_min)
            
    # Step 4: Convert Join Tree to Contour Tree
    # Leaves of contour tree are local extrema (maxima and minima)
    # Interior nodes are saddles
    contour_tree = convert_to_contour_tree(join_tree)
    
    # Step 5: Optional Augmentation
    # Add non-branching vertices at non-critical points for seeding
    contour_tree = augment_contour_tree(contour_tree, scalar_field)
    
    return contour_tree

def find_critical_points(scalar_field, domain):
    """
    Identifies local maxima, minima, and saddle points.
    Implementation depends on discretization (grid vs mesh).
    """
    # [G] Standard approach: compare scalar value of a vertex to its neighbors.
    # If value > all neighbors -> local max.
    # If value < all neighbors -> local min.
    # Otherwise -> saddle.
    # Neighbors are defined by the domain connectivity (e.g., 6-connectivity for 3D grid).
    # Changing neighbor definition slightly alters critical point locations 
    # but not the tree structure size [S1, unofficial].
    pass

def sort_by_scalar_value(critical_points, descending=True):
    """
    Sorts critical points by their scalar value.
    """
    return sorted(critical_points, key=lambda p: p.scalar_value, reverse=descending)

def get_components_affected_by(saddle_point, scalar_field):
    """
    Determines which connected components of the sub-level set 
    are affected by the saddle point.
    """
    # [G] This requires tracking connected components as the isovalue 
    # passes through the saddle's scalar value.
    pass

def convert_to_contour_tree(join_tree):
    """
    Converts the merge tree structure into the contour tree representation.
    """
    # The contour tree structure mirrors the merge tree but explicitly 
    # distinguishes between split and merge saddles and maps leaves 
    # to both maxima and minima.
    pass

def augment_contour_tree(contour_tree, scalar_field):
    """
    Adds non-branching vertices at non-critical points.
    """
    # [G] Insert vertices along edges where they can serve as seeds 
    # for specific isovalue contours.
    pass
```

**Parameters and Typical Values:**
*   **Neighbor Definition:** The method of defining neighbors during split/merge tree computations can slightly alter the location of critical points, though the underlying structure (number of supernodes and superarcs) remains fixed for a given input [S1, unofficial].
*   **Seeds:** Using seeds significantly reduces the number of seeds required for representation but often increases the overall storage size of the contour tree [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Non-Simply Connected Domains:** The contour tree does not correctly handle data without a boundary or with loops (genus). The input domain must be simply connected [S1, unofficial]. For such cases, use the [Reeb Graph](9315-reeb-graph.md) [S1, unofficial].
*   **High Dimensionality:** Critical vertices are difficult to identify in data with dimensionality of four or more [S1, unofficial].
*   **Genus Information:** The contour tree typically does not contain information regarding changes in topological genus; this must be added to the output separately if needed [S1, unofficial].
*   **Load Imbalance:** Multithreaded approaches (e.g., contour forest) may suffer from load imbalance and redundant computations, limiting speedup [S1, unofficial].

## Subtopics

### [Reeb Graph](9315-reeb-graph.md)
Generalizes the contour tree to handle domains with loops or voids by tracking level sets modulo homotopy. Choose this when the input domain is not simply connected.

### [Joint Contour Net](9316-joint-contour-net.md)
Extends contour analysis to multivariate scalar fields. Choose this when analyzing multiple scalar fields simultaneously.

## Operating characteristics
*   **Time Complexity:** Not explicitly given in the sources. Chiang et al. improve processing time by only sorting critical vertices rather than all elements, vastly increasing speed in very large data sets [S1, unofficial].
*   **Memory Complexity:** Not explicitly given. However, using seeds often increases the overall storage size of the contour tree compared to other methods [S1, unofficial].
*   **Measured Performance:** Gueunet et al.'s multithreaded "contour forest" algorithm achieves quicker computations, but speedup is limited by load imbalance and redundant computations [S1, unofficial].
*   **Strengths:** Simple method for capturing topological information; one-to-one correspondence with contours; useful for feature extraction and analysis [S1, unofficial].
*   **Limitations:** Cannot handle non-simply connected domains (loops/voids); critical point identification is difficult in 4D+; does not inherently track genus changes [S1, unofficial].
*   **When to Use:** For analyzing topology of scalar fields on simply connected domains, feature extraction, and as a building block for more complex structures like the Reeb Graph [S1, unofficial].
*   **When Not to Use:** When the domain has loops or voids (use Reeb Graph); when analyzing multivariate data (use Joint Contour Net); in 4D+ data where critical point identification is unreliable [S1, unofficial].

## Background
The contour tree was introduced by Boyell and Ruston [S1, unofficial]. It uses concepts from Morse theory to capture changes in the topology of scalar fields at critical points [S1, unofficial]. Merge trees, which are fundamental to constructing contour trees, are used to compute zero-dimensional persistence measures representing the number of connected components in a region [S1, unofficial]. The contour tree is often used as a computation step in the construction of more complex topological data structures [S1, unofficial].

## Sources
[S1] Chapter Topological Visualisation Techniques for Volume Multifield Data, https://archive.org/details/oapen-20.500.12657-49310, fetched 2026-10-27 (unofficial)
