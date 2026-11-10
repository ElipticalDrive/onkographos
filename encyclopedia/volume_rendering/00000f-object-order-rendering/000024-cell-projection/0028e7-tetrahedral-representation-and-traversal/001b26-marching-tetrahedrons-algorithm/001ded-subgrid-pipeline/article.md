# Subgrid Pipeline

Also called: Subgrid Marching Pipeline

## Overview
The Subgrid Pipeline is a method for extracting isosurfaces from tetrahedral grids by resolving topological ambiguities within individual tetrahedra using subgrid-accurate intersection data [S1, unofficial][S2]. It distinguishes itself from standard Marching Tetrahedrons by explicitly handling open, scoop, and normal boundary curves to ensure watertightness, allowing multiple intersection vertices per edge to represent complex topology [S1, unofficial]. It builds on [Marching Tetrahedrons Algorithm](940-marching-tetrahedrons-algorithm.md) by replacing the fixed lookup table with a local reconstruction procedure that processes boundary curves [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
- **Input:** A range of tetrahedra in a grid, a scalar field defined on vertices, an isovalue, and configuration options (`mod2`, `greedy`, `compute_normals`).
- **Output:** A global `TriangleSoup` containing the extracted surface triangles.

**Data Structures:**
- `EdgeInts`: A structure storing intersection data for the 6 edges of a tetrahedron.
  - Fields: `ne_01`, `ne_02`, `ne_03`, `ne_12`, `ne_13`, `ne_23`.
  - Each `ne_ij` represents the number of intersections on edge $(i,j)$. In the standard pipeline, these are integers derived from t-values. In the `Mod2 Reduction` variant, they are reduced modulo 2.
- `CombFace`: Represents a boundary curve (open, scoop, or normal). Contains a list of vertices defining the curve.
- `TriangleSoup`: A collection of triangles (triplets of vertices) accumulated from all tetrahedra.

**Steps:**
1. **Iterate Tetrahedra:** Loop through each tetrahedron in the input range using a `TetRange` iterator [S1, unofficial].
2. **Query Intersections:** For the current tetrahedron, query the edge intersections.
   - Request t-values (positions) and normals for all 6 edges if `compute_normals` is true [S1, unofficial].
   - Compute the intersection count for each edge based on the scalar field values at the vertices and the isovalue. Let $N_{ij}$ be the number of intersections on edge $(i,j)$.
3. **Mod2 Reduction (Optional):** If the `mod2` option is enabled:
   - Apply modulo-2 reduction to the intersection counts: $N'_{ij} = N_{ij} \pmod 2$ [S1, unofficial].
   - This limits the output to at most one intersection per edge, restoring the single-surface-per-edge constraint of classic Marching Tetrahedra while using subgrid-accurate vertex positions [S1, unofficial].
   - If the reduction results in a configuration that implies no surface (e.g., all $N'_{ij} = 0$), skip processing this tetrahedron [S1, unofficial].
4. **Construct Boundary Curves:** Call `boundary_comb_curves` with the tetrahedron indices, intersection counts, and edge occupation flags [S1, unofficial].
   - This function analyzes the intersection topology to identify three types of curves:
     - `open_curves`: Curves that enter and exit the tetrahedron.
     - `scoop_curves`: Curves that form closed loops or "scoops" within the tetrahedron.
     - `normal_curves`: Curves associated with normal boundary conditions.
   - The function returns these three lists of `CombFace` objects [S1, unofficial].
5. **Track Statistics (Optional):**
   - Increment a counter for "non-even" tetrahedra if `open_curves` is not empty [S1, unofficial].
   - Increment a counter for "non-normal" tetrahedra if `open_curves` is empty AND `scoop_curves` is empty [S1, unofficial].
6. **Construct Local Surface:**
   - Create a local `TriangleSoup` for the current tetrahedron.
   - Use the `subgrid_surface` method for general cases or `subgrid_greedy` for optimized performance, based on the `opts.greedy` flag [S1, unofficial].
   - The reconstruction procedure resolves the boundary curves into triangles, ensuring watertightness [S1, unofficial].
7. **Merge:** Append the local `TriangleSoup` to the global result soup [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def subgrid_pipeline(tet_range, scalar_field, isovalue, opts):
    global_soup = TriangleSoup()
    
    for tet in tet_range:
        # Step 2: Query edge intersections
        edge_ints = EdgeInts()
        for edge_idx in range(6):
            edge = tet.edges[edge_idx]
            # Compute intersection count N_ij based on scalar values and isovalue
            # This may involve solving for t-values if multiple intersections exist
            n_ij = compute_edge_intersections(edge, scalar_field, isovalue)
            edge_ints.set(edge_idx, n_ij)
            
            # If normals are needed, compute and store them here
            if opts.compute_normals:
                normals[edge_idx] = compute_normal(edge, scalar_field, isovalue)
        
        # Step 3: Mod2 Reduction
        if opts.mod2:
            for edge_idx in range(6):
                edge_ints.set(edge_idx, edge_ints.get(edge_idx) % 2)
            
            # Check if all intersections are zero after mod2
            if all(edge_ints.get(i) == 0 for i in range(6)):
                continue
        
        # Step 4: Construct boundary curves
        open_curves, scoop_curves, normal_curves = boundary_comb_curves(
            tet.indices, edge_ints, opts
        )
        
        # Step 5: Track statistics (optional)
        if opts.track_stats:
            if open_curves:
                non_even_count += 1
            if not open_curves and not scoop_curves:
                non_normal_count += 1
        
        # Step 6: Construct local surface
        if opts.greedy:
            local_soup = subgrid_greedy(tet, open_curves, scoop_curves, normal_curves, edge_ints)
        else:
            local_soup = subgrid_surface(tet, open_curves, scoop_curves, normal_curves, edge_ints)
        
        # Step 7: Merge
        global_soup.merge(local_soup)
    
    return global_soup
```

**Parameters:**
- `mod2`: Boolean. If true, reduces intersection counts modulo 2 [S1, unofficial].
- `greedy`: Boolean. If true, uses the optimized `subgrid_greedy` reconstruction [S1, unofficial].
- `compute_normals`: Boolean. If true, computes normals for intersection points [S1, unofficial].
- `track_stats`: Boolean. If true, collects statistics on curve types [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Subgrid methods require accurate computation of intersection positions. Floating-point errors can lead to non-watertight surfaces if not handled carefully [S2].
- **Degenerate Input:** Tetrahedra with zero volume or collinear vertices may cause issues in intersection computation.
- **Complex Topology:** The ability to handle multiple intersections per edge allows for complex topologies, but increases computational cost [S2].

## Subtopics

### [Mod2 Reduction](1665-mod2-reduction.md)
Mod2 Reduction is an ablation of the Subgrid Marching Tetrahedra method that provides exact input intersection locations but limits the output to at most one intersection per edge. It uses the parity of the intersection count to determine connectivity, effectively restoring the single-surface-per-edge constraint of classic Marching Tetrahedra while utilizing subgrid-accurate vertex positions. It is distinguished from its sibling [Subgrid Surface Reconstruction](1666-subgrid-surface-reconstruction.md) by its strict limitation to one output vertex per edge, whereas the full subgrid method allows multiple vertices to represent complex topology.

### [Subgrid Surface Reconstruction](1666-subgrid-surface-reconstruction.md)
Subgrid Surface Reconstruction is a method for extracting a triangle soup from a tetrahedral grid by processing individual tetrahedra to resolve topological ambiguities in isosurface extraction. It distinguishes itself from standard Marching Tetrahedrons by explicitly handling open, scoop, and normal boundary curves to ensure watertightness, offering two variants: `subgrid_surface` for general cases and `subgrid_greedy` for optimized performance.

### Subgrid Marching Triangles
Subgrid Marching Triangles is a 2D contouring algorithm that reconstructs curves within a single triangle using a subgrid procedure, applied to all triangles in a 2D triangular grid given intersection counts on grid edges [S2]. Unlike 3D subgrid marching tets, this 2D variant does not need to worry about compatibility between neighboring cells, allowing for the specific modification in the odd sum case: rather than subtracting 1 from all 3 edge coordinates in a face, subtract 1 from the largest edge coordinate, breaking ties arbitrarily [S2]. For the dual method, it uses the same regularized QEF as in Section 3.3.2 of [S2], but in two dimensions. It can be applied to arbitrary unstructured triangulations, including surface meshes, as it makes no assumptions about the structure of the triangle grid [S2].

## Operating characteristics
- **Time Complexity:** The per-tetrahedron cost is greater than that of marching tets because it may produce more polygons per tet and involves additional logic for local reconstruction [S2]. However, the amortized cost per output triangle is quite comparable [S2].
- **Memory Complexity:** Depends on the number of output triangles. The method requires storing boundary curves and local triangle soups during processing.
- **Performance:** Speed improvements can be achieved by precomputing a table of common stencils if speed or thread synchronization is critical [S2].
- **Strengths:** Handles complex topology with multiple intersections per edge; ensures watertightness; applicable to arbitrary unstructured grids [S1, unofficial][S2].
- **Limitations:** Higher per-tetrahedron computational cost compared to classic marching tets; potential for increased output polygon count [S2].
- **When to Use:** When topological accuracy and watertightness are critical, and the grid is unstructured or contains complex features [S1, unofficial][S2].
- **When Not to Use:** When performance is the primary concern and the topology is simple enough for classic marching tets [S2].

## Background
The Subgrid Pipeline originates from the work on Subgrid Marching Tetrahedra, which aims to improve the accuracy of isosurface extraction by considering subgrid details within each tetrahedron [S2]. The method was developed to address the limitations of classic Marching Tetrahedrons in handling complex topologies and ensuring watertightness [S1, unofficial][S2]. The 2D variant, Subgrid Marching Triangles, extends these concepts to two-dimensional contouring [S2].

## Sources
[S1] hbaktash/subgrid-marching, https://github.com/hbaktash/subgrid-marching, fetched 2026-10-27 (unofficial)
[S2] Subgrid Marching Tetrahedra, https://doi.org/10.1145/3811358, fetched 2026-10-26
