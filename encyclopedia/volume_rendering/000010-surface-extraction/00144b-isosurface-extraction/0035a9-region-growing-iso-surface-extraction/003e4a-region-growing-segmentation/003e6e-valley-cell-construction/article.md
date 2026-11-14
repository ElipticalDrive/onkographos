# Valley Cell Construction

Also called: Region growing from maxima, Valley cell decomposition

## Overview
Valley Cell Construction decomposes a feature space into non-overlapping regions called valley cells, where each cell contains exactly one isolated local maximum [S1]. It structures the feature space for clustering or segmentation by identifying the boundaries between basins of attraction for local maxima [S1]. It differs from [Graph-based Isosurface Segmentation (Min-Cut)](9748-graph-based-isosurface-segmentation-min-cut.md) by using topological region growing from maxima rather than global cut optimization, and from [Restricted Region Growing](9750-restricted-region-growing.md) by explicitly handling topological degeneracies (multi-maximum saddles) via node duplication to ensure a valid cell decomposition [G].

## How it works
**Inputs:** A discretized feature space represented as a 2D triangular mesh with scalar values at vertices [S1]. The mesh enforces a piecewise linear approximation of the density function [S1].
**Outputs:** A set of valley cells, each associated with a unique maximum, and a map of saddle correspondences including persistence values [S1].

**Data Structures:**
*   **Mesh:** Triangular mesh $T$. Each triangle has 3 vertex indices.
*   **Vertices:** Store scalar value $v$.
*   **Saddles:** Identified critical points. Store a set of associated maxima $\mathcal{M}_S$.
*   **Cells:** Set of triangles belonging to a specific maximum.

**Steps:**

1.  **Identify Critical Points:** Identify local maxima, minima, and saddles on the mesh. A saddle is an extreme where two ridge lines and two valley lines meet [S1].
2.  **Region Growing:**
    *   Initialize a cell for each maximum $m$ [S1].
    *   For each maximum, recursively add neighboring triangles to its cell [S1].
    *   **Stopping Condition:** Stop growing a cell when the boundary reaches a valley line or a triangle whose vertices are all minima [S1].
    *   During growth, track the correspondence between the growing cell and the saddles encountered on its boundary [S1].
3.  **Saddle Update and Persistence Calculation:**
    *   After growing, each saddle $S$ has a set of connected maxima $\mathcal{M}_S$ [S1].
    *   Calculate the **persistence** of saddle $S$ relative to a maximum $m \in \mathcal{M}_S$ as the difference in scalar values:
        $$ p(S, m) = v(m) - v(S) $$
    *   The persistence of the saddle $S$ is defined as the minimum difference between the saddle value and its associated maximum values:
        $$ p(S) = \min_{m \in \mathcal{M}_S} (v(m) - v(S)) $$
4.  **Handle Degeneracies (Saddle Duplication):**
    *   In degenerate cases (e.g., multiple valley lines meeting at a saddle, or paths ending at a junction), a single saddle may belong to more than two cells (associated with $k > 2$ maxima) [S1].
    *   To resolve this, duplicate the saddle node $k-1$ times [S1].
    *   Replace the original saddle $S$ with $k$ distinct saddle instances $S_1, S_2, \dots, S_k$.
    *   Each instance $S_i$ is associated with exactly one maximum from $\mathcal{M}_S$ [S1].
    *   *Note:* This is similar to unfolding a $k$-fold saddle but only duplicates the node, not the paths [S1].
5.  **Boundary Persistence Calculation:**
    *   Each boundary between two adjacent valley cells is associated with at least one saddle [S1].
    *   Calculate the **boundary persistence** as the minimum persistence of all saddles associated with that boundary:
        $$ p_{\text{boundary}} = \min_{S \in \text{Boundary}} p(S) $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def construct_valley_cells(mesh):
    # mesh: list of triangles, each triangle is a tuple of 3 vertex indices
    # vertices: list of dicts with 'value'
    
    # Precompute triangle adjacency
    adj = {i: set() for i in range(len(mesh))}
    for i in range(len(mesh)):
        for j in range(i + 1, len(mesh)):
            if len(set(mesh[i]) & set(mesh[j])) == 2:
                adj[i].add(j)
                adj[j].add(i)
                
    # 1. Identify maxima [G]
    maxima = [v for v in range(len(vertices)) if all(vertices[v]['value'] >= vertices[n]['value'] for n in get_vertex_neighbors(v, mesh))]
    
    # 2. Region Growing
    cells = {m: set() for m in maxima}
    saddle_max_map = {} # saddle_idx -> set of max_indices
    
    for m in maxima:
        start_tris = [i for i, tri in enumerate(mesh) if m in tri]
        stack = list(start_tris)
        visited = set()
        
        while stack:
            tri_idx = stack.pop()
            if tri_idx in visited: continue
            visited.add(tri_idx)
            cells[m].add(tri_idx)
            
            tri = mesh[tri_idx]
            # Stopping condition: triangle whose vertices are all minima
            if all(vertices[v]['value'] == min(vertices[n]['value'] for n in get_vertex_neighbors(v, mesh)) for v in tri):
                continue
                
            # Check boundary edges
            for i in range(3):
                edge = (tri[i], tri[(i+1)%3])
                # Check if edge is shared with an unvisited triangle
                shared = False
                for n_idx in adj.get(tri_idx, []):
                    if set(edge).issubset(set(mesh[n_idx])):
                        if n_idx not in visited:
                            stack.append(n_idx)
                        shared = True
                        break
                if not shared:
                    # Boundary edge: check if valley line or saddle
                    if is_valley_line(edge, vertices):
                        saddle_idx = get_saddle_at_edge(edge, mesh)
                        if saddle_idx is not None:
                            saddle_max_map.setdefault(saddle_idx, set()).add(m)
                            
    # 3. Update Saddle Correspondences and Persistence
    saddles = [s for s in range(len(vertices)) if is_saddle(s, vertices, mesh)]
    for s in saddles:
        if s not in saddle_max_map:
            saddle_max_map[s] = set()
        pers_values = {m: vertices[m]['value'] - vertices[s]['value'] for m in saddle_max_map[s]}
        saddles[s].persistence = min(pers_values.values()) if pers_values else 0.0
        saddles[s].associated_maxima = list(saddle_max_map[s])
        
    # 4. Resolve Degeneracies via Duplication
    new_saddles = []
    for s in saddles:
        k = len(s.associated_maxima)
        if k > 2:
            for i in range(1, k):
                new_s = copy(s)
                new_s.associated_maxima = [s.associated_maxima[i]]
                new_s.persistence = vertices[s.associated_maxima[i]]['value'] - vertices[s]['value']
                new_saddles.append(new_s)
            s.associated_maxima = [s.associated_maxima[0]]
            s.persistence = vertices[s.associated_maxima[0]]['value'] - vertices[s]['value']
            new_saddles.append(s)
        else:
            new_saddles.append(s)
            
    # 5. Calculate Boundary Persistence
    boundaries = find_adjacent_cell_boundaries(cells, mesh)
    for b in boundaries:
        b.persistence = min(s.persistence for s in b.saddles) if b.saddles else 0.0
        
    return cells, new_saddles, boundaries
```

## Subtopics

### [Valley Surface Extraction via Point Seeding and Convergence](10401-valley-surface-extraction-via-point-seeding-and-convergence.md)
A mesh-free method that grows a surface by seeding points on an adaptive ellipsoid around existing surface points and converging them to the valley surface (maximal convexity ridge) using polynomial approximation or Newton iteration.

## Operating characteristics
*   **Complexity:** Linear with respect to the number of triangles in the mesh for the region growing phase, assuming efficient neighbor traversal [G]. Saddle duplication is proportional to the number of degenerate saddles [S1].
*   **Strengths:** Provides a clear topological decomposition of the feature space. Explicitly handles noise and discretization artifacts through boundary persistence, allowing for cell simplification [S1].
*   **Limitations:** Degenerate cases (multi-fold saddles, junctions) require special handling (duplication) [S1]. The method relies on the quality of the underlying mesh and critical point identification [S1].
*   **Use Cases:** Automating transfer function design by clustering 2D density plots; isolating semantic features in volume data [S1].

## Background
The method is inspired by the Morse complex construction scheme described in [EHZ03]. It was formalized for automating transfer function design in the context of 2D density plots for volume exploration [S1]. The approach uses region growing from maxima [HP74] to tessellate the feature space into valley cells, ensuring each cell contains an isolated maximum [S1].

## Sources
[S1] Automating Transfer Function Design with Valley Cell‐Based Clustering of 2D Density Plots, https://doi.org/10.1111/j.1467-8659.2012.03122.x, fetched 2026-11-01
