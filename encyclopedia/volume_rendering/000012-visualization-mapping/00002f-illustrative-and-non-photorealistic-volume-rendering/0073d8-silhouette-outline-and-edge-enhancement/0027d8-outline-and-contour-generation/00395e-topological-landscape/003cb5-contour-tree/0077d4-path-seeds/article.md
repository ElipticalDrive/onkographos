# Path Seeds

Also called: First Steps

## Overview
Path seeds are a method for generating isosurface seeds directly from a contour tree at runtime, replacing pre-computed minimal seed sets. This approach reduces memory overhead and pre-processing time while maintaining a one-to-one correspondence between contours, isosurface seeds, and superarcs of the contour tree [S1]. It sets itself apart from minimal seed sets (van Kreveld et al.) by storing only the initial edge from critical points, achieving provably minimal storage complexity of $\Theta(t)$ where $t$ is the number of branches, rather than potentially $\Theta(n)$ [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A scalar field defined on a mesh (vertices and edges), the contour tree (containing critical points and superarcs).
*   **Output:** A set of "path seeds" (edges) stored within the contour tree structure, which can be used to generate specific isosurface contours on demand.

**Data Structures**
The method integrates with the standard contour tree construction algorithm (Carr, Snoeyink & Axen [S1]). The key data structure is the **Join Tree** and **Split Tree**, which are merged to form the **Contour Tree**.
*   **Join Tree:** Constructed by sweeping from high to low isovalues. It uses a Union-Find structure to track connected components of the super-level set $\{x : f(x) \geq h\}$.
*   **Split Tree:** Constructed similarly (sweeping low to high or identifying splits). It tracks connected components of the sub-level set.
*   **Path Seed Storage:** For each superarc in the contour tree, the algorithm stores a single edge, referred to as the "first step" or "path seed." This edge originates at a critical point (local extremum or saddle) and ascends into the first non-critical vertex along the monotone path corresponding to that superarc.

**Steps**
1.  **Construct Join Tree:**
    *   Sweep isovalues from high to low.
    *   Use Tarjan’s Union-Find structure to determine connected components of the super-level set $\{x : f(x) \geq h\}$ [S1].
    *   When a vertex $v$ is added to the Union-Find, identify edges ascending from $v$ into the connected components that are merging.
    *   Specifically, if adding $v$ causes two previously disjoint components to merge, identify the edges from $v$ into these components.
    *   Store these edges as path seeds in the Join Tree [S1].
2.  **Construct Split Tree:**
    *   Sweep isovalues from low to high (or identify splits).
    *   Identify split points where components separate.
    *   Store ascending edges from split points as path seeds in the Split Tree [S1].
3.  **Merge Trees:**
    *   Merge the Join Tree and Split Tree to obtain the Contour Tree.
    *   Transfer the stored path seeds from the Join and Split Trees to the corresponding superarcs in the Contour Tree [S1].
4.  **Runtime Extraction:**
    *   To extract a contour for a specific isovalue $h$ along a superarc:
    *   Start at the stored path seed edge (the "first step") originating from the critical point associated with that superarc.
    *   Follow the path through the mesh (ascending or descending as defined by the superarc) until the desired isovalue $h$ is reached [S1].

**Formulas and Definitions**
*   **Path Seed:** The first edge $(v, u)$ from a critical point $v$ to a non-critical vertex $u$ along a monotone path corresponding to a superarc [S1].
*   **Storage Complexity:** $\Theta(t)$, where $t$ is the number of supernodes/branches in the contour tree. This is provably minimal because distinct seeds are required for each branch at any fork in the contour tree [S1].
*   **Construction Cost:** $O(n \log n + t\alpha(t))$, where $n$ is the number of vertices and $\alpha(t)$ is the inverse Ackermann function. This is the same asymptotic cost as the contour tree algorithm itself, with constant cost per edge for seed extraction [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_path_seeds(mesh, scalar_field):
    """
    Generates path seeds by integrating seed extraction with contour tree construction.
    Returns a contour tree where each superarc is annotated with a path seed edge.
    """
    # Initialize Union-Find structure for Join Tree construction
    # Union-Find supports: find_set(x), union_sets(x, y)
    union_find = UnionFind(mesh.vertices)
    
    # Sort vertices by scalar value for sweeping
    # High to low for Join Tree (super-level sets)
    sorted_vertices_high_to_low = sort_by_value_desc(mesh.vertices, scalar_field)
    
    join_tree_seeds = {} # Map superarc_id -> seed_edge
    
    # Construct Join Tree
    for vertex in sorted_vertices_high_to_low:
        v = vertex
        v_value = scalar_field[v]
        
        # Identify neighbors of v that have higher values (already processed)
        # These neighbors represent the connected components merging at v
        higher_neighbors = [u for u in mesh.neighbors(v) if scalar_field[u] > v_value]
        
        # Group higher neighbors by their current Union-Find set
        # Each set represents a connected component in the super-level set
        components = {}
        for u in higher_neighbors:
            root = union_find.find_set(u)
            if root not in components:
                components[root] = []
            components[root].append(u)
        
        # If there are multiple components, v is a join point
        # The edges from v to the first vertex in each component are path seeds
        if len(components) > 1:
            seed_edges = []
            for root, neighbor_list in components.items():
                # Pick one neighbor from each component to form the seed edge
                # The edge is (v, neighbor)
                seed_edges.append((v, neighbor_list[0]))
            
            # Store these edges as path seeds for the corresponding superarcs
            # In the actual algorithm, these are associated with the specific branches
            # merging at v. Here we store them in a temporary structure to be
            # transferred to the contour tree later.
            join_tree_seeds[v] = seed_edges
        
        # Union the vertex with its higher neighbors
        for u in higher_neighbors:
            union_find.union_sets(v, u)

    # Construct Split Tree similarly (sweeping low to high)
    # ... (omitted for brevity, symmetric logic) ...
    
    # Merge Join and Split Trees to form Contour Tree
    # Transfer seeds from join_tree_seeds and split_tree_seeds to the
    # corresponding superarcs in the contour tree.
    contour_tree = merge_join_split_trees(join_tree_seeds, split_tree_seeds)
    
    return contour_tree

def extract_contour(contour_tree, superarc_id, target_isovalue):
    """
    Extracts a contour for a specific isovalue using the path seed.
    """
    # Retrieve the path seed (first step edge) for the superarc
    seed_edge = contour_tree.get_seed_edge(superarc_id)
    start_vertex = seed_edge[0] # Critical point
    current_vertex = seed_edge[1] # First non-critical vertex
    
    # Follow the path from the seed edge until target_isovalue is reached
    # The path direction is determined by the superarc (ascending/descending)
    path = [seed_edge]
    current = current_vertex
    
    while not isovalue_reached(current, target_isovalue):
        # Move to the next vertex along the monotone path
        # In a mesh, this involves finding the neighbor that continues
        # the ascent/descent towards the critical point defining the superarc.
        next_vertex = get_next_vertex_in_path(current, target_isovalue, contour_tree)
        path.append((current, next_vertex))
        current = next_vertex
        
    return path
```

**Parameters and Edge Cases**
*   **Path Choice:** The path need not extend to a local extremum, need not take the steepest ascent, and need not terminate at a critical point [S1]. Any monotone path from the critical point along the superarc is valid because all ascending edges from a non-critical vertex pass through the same set of contours [S1].
*   **Precision:** The method relies on exact topological correspondence. Degenerate cases (e.g., flat regions) must be handled by the underlying contour tree construction algorithm [S1].
*   **Runtime Cost:** While storage is minimal, extracting the full contour on demand can cost $\Omega(n)$ in the worst case for triangulation, though this is typically dwarfed by the cost of the actual contour extraction $O(N^{(d-1)/d})$ [S1].

## Operating characteristics
*   **Time Complexity:** Asymptotic construction cost is $O(n \log n + t\alpha(t))$, identical to the contour tree algorithm [S1]. Runtime extraction cost depends on the path length but is generally efficient for visualization purposes [S1].
*   **Memory Complexity:** $\Theta(t)$, where $t$ is the number of branches in the contour tree. This is provably minimal and significantly cheaper than minimal seed sets, which can be $\Theta(n)$ in higher dimensions [S1].
*   **Strengths:** Reduces memory overhead and pre-processing time. Guarantees a 1-1 correspondence between contours, seeds, and superarcs, enabling individual contour tracking and annotation [S1].
*   **Limitations:** Requires the contour tree to be constructed first. Runtime extraction adds a small overhead compared to pre-computed seeds, though this is negligible compared to full isosurface extraction [S1].

## Background
Path seeds were introduced in the context of exploratory visualization using topology. The method was proposed to replace the minimal seed set approach of van Kreveld et al. [S1]. It integrates seed generation directly with the contour tree construction algorithm of Carr, Snoeyink & Axen [S1]. The technique enables "flexible isosurfaces," allowing users to select, manipulate, and annotate individual contours at different isovalue levels, generalizing the concept of a single isosurface [S1].

## Sources
[S1] Path Seeds and Flexible Isosurfaces Using Topology for Exploratory Visualization, https://doi.org/10.2312/vissym/vissym03/049-058, fetched 2026-10-30
