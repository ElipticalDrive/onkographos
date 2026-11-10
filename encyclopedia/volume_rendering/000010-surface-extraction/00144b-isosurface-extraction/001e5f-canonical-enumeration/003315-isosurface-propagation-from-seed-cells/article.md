# Isosurface Propagation from Seed Cells

Also called: Seed Set / Surface Propagation, Surface Propagation, seed-based extraction

## Overview
Isosurface Propagation from Seed Cells is a geometric decomposition technique that extracts isosurfaces by identifying a sparse set of "seed" cells containing the surface and propagating the extraction to neighboring cells [S1]. It combines the spatial coherence exploitation of span space algorithms with the structural awareness of geometric decomposition [S1]. Unlike [Canonical Enumeration](1779-canonical-enumeration.md), which visits every cell, this method skips empty regions by traversing a hierarchical search structure of seeds, theoretically offering near-optimal worst-case time complexity [S1].

## How it works
**Inputs:** A scalar volume dataset $V$ defined on a structured or unstructured mesh, and an isovalue $v$.
**Outputs:** A polygonal mesh representing the isosurface $V(x) = v$.

**Data Structures:**
1.  **Seed Set ($S$):** A collection of cells known to intersect the isosurface.
2.  **Hierarchical Search Structure:** A tree or graph organizing the seed set to allow efficient traversal and neighbor lookup. The specific implementation varies by variant (e.g., segment tree, contour tree, or spatial graph) [S1].
3.  **Adjacency Map:** Precomputed or on-the-fly mapping of cell neighbors to enable propagation [S1].
4.  **Edge Hash Table (Optional):** Used to exploit spatial coherence by caching edge interpolation results, preventing recomputation for cells sharing an edge [S1].

**Algorithm Steps:**

1.  **Seed Identification:** Locate cells that intersect the isosurface. This is the most complex phase. Common strategies include:
    *   **Itoh et al.:** Find local extremum points in the data. Connect them with a graph in the spatial domain. The seed set includes cells containing these extremum points, all cells intersected by the graph arcs, and boundary cells if the volume has holes. A thinning algorithm is applied to generate a skeleton of the seed set, reducing the number of cells while maintaining connectivity [S1].
    *   **van Kreveld et al.:** Construct a graph of local extremum points and add saddle points to create a contour tree [S1].
    *   **Bajaj et al.:** Use set theory to identify seed cells and organize them in a segment tree for traversal [S1].
2.  **Validation:** Ensure the seed set contains at least one cell per connected component of each isosurface [S1].
3.  **Propagation:**
    *   Initialize the extraction at the identified seed cells.
    *   Traverse the hierarchical search structure to find all seeds intersecting the current isovalue [S1].
    *   For each active seed cell, extract the polygonal geometry (e.g., using Marching Cubes or a similar local extractor).
    *   Propagate to neighboring cells using adjacency information. If a neighbor is not in the seed set but intersects the isosurface, add it to the processing queue and repeat [S1].
    *   Use the edge hash table to cache edge interpolations. If a cell shares an edge with a previously processed cell, retrieve the interpolation from the hash entry instead of recomputing it [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_isosurface_seed_propagation(volume, isovalue):
    # Step 1: Identify Seed Cells
    # Strategy depends on variant (Itoh, van Kreveld, Bajaj)
    # Here we assume a generic function that returns a set of seed cells
    seed_cells = identify_seeds(volume, isovalue)
    
    # Step 2: Validate Seed Set
    # Ensure at least one cell per connected component
    # [G] Implementation detail: Connectivity check via BFS/DFS on seed adjacency
    if not is_seed_set_complete(seed_cells, volume):
        raise ValueError("Seed set does not cover all connected components")

    # Step 3: Initialize Structures
    processed_cells = set()
    queue = list(seed_cells)
    mesh = PolygonMesh()
    edge_cache = HashTable() # Maps edge_id -> interpolated_point

    # Step 4: Propagate
    while queue is not empty:
        cell = queue.pop()
        
        if cell in processed_cells:
            continue
        processed_cells.add(cell)

        # Extract geometry for this cell
        # [G] Local extraction logic (e.g., Marching Cubes)
        cell_mesh = extract_cell_geometry(cell, volume, edge_cache)
        mesh.add(cell_mesh)

        # Propagate to neighbors
        neighbors = get_adjacent_cells(cell)
        for neighbor in neighbors:
            if neighbor not in processed_cells:
                # Check if neighbor intersects isosurface
                # [G] Min/Max value check or precise intersection test
                if intersects_isosurface(neighbor, volume, isovalue):
                    queue.append(neighbor)

    return mesh

def identify_seeds(volume, isovalue):
    # Placeholder for specific seed identification algorithms
    # Itoh: Extrema + Graph + Thinning
    # van Kreveld: Contour Tree
    # Bajaj: Set Theory + Segment Tree
    # [G] This function is highly variable and not fully specified in sources
    return set()
```

**Parameters:**
*   **Thinning Algorithm:** Used in Itoh's method to reduce seed count. Typical parameters depend on the image processing library used. Not given in the sources.
*   **Hash Table Size:** Depends on the number of edges. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Noisy Data:** Measurement data (MRI/CT) can cause the seed set construction to produce a large number of seed cells, significantly slowing preprocessing time [S1].
*   **Branch Prediction:** The surface propagation code contains a large number of branches. Processors cannot readily predict the outcome, leading to poor performance on some hardware [S1].
*   **Holes:** If the volume has holes, boundary cells must be explicitly included in the seed set to ensure connectivity [S1].

## Subtopics

### Itoh's Seed Set Method
Uses local extrema and graph connectivity with thinning. Best for smooth data where extrema are well-defined. [S1]

### Contour Tree Based Propagation
Uses van Kreveld's contour tree approach. Theoretically robust for complex topologies but implementation details are sparse. [S1]

### Segment Tree Based Propagation
Uses Bajaj's set theory and segment tree. Offers efficient traversal but requires complex data structure management. [S1]

## Operating characteristics
*   **Time Complexity:** Theoretically near-optimal worst-case time complexity [S1]. However, practical performance is heavily influenced by the efficiency of seed identification and the branch prediction penalty during propagation [S1].
*   **Memory Complexity:** Memory overhead is nearly as much as the NOISE algorithm due to the extrema data structure used [S1]. This is significantly higher than Branch-on-Need Octree (BONO), which retains the original representation with minimal overhead [S1].
*   **Performance:** On the Rage256 dataset, the surface propagation algorithm performed poorly compared to BONO, ISSUE, and NOISE due to branch misprediction [S1]. On the Head 128 dataset, it showed competitive results but was still slower than the best-performing algorithms [S1].
*   **Strengths:** Exploits spatial coherence; handles structured and unstructured meshes; theoretically optimal complexity.
*   **Limitations:** High preprocessing time for noisy data; poor branch prediction performance; high memory overhead compared to octree methods.
*   **When to Use:** When the data is smooth and has few connected components, and when memory is not the primary constraint.
*   **When Not to Use:** For noisy medical data (MRI/CT) where seed identification becomes expensive, or on hardware with poor branch prediction capabilities.

## Background
The method was introduced as a geometric decomposition technique that combines aspects of span space algorithms and geometric decomposition [S1]. Key contributors include Itoh et al. [6,7], van Kreveld et al. [16], and Bajaj et al. [1]. It was evaluated in a case study comparing various 3D isosurface extraction techniques, including Marching Cubes, BONO, NOISE, ISSUE, and Interval Tree [S1]. The method is applicable to both structured and unstructured meshes [S1].

## Sources
[S1] A Case Study of Isosurface Extraction Algorithm Performance, https://doi.org/10.1007/978-3-7091-6783-0_26, fetched 2026-10-26
