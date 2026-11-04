# Subgrid Pipeline

Also called: Subgrid Marching Pipeline

## Overview
The Subgrid Pipeline is a method for extracting watertight triangle soups from tetrahedral grids by explicitly resolving topological ambiguities in isosurface extraction [S1, unofficial]. It distinguishes itself from standard Marching Tetrahedrons by processing individual tetrahedra to generate open, scoop, and normal boundary curves, ensuring topological correctness where standard algorithms fail [S1, unofficial].

## How it works
The pipeline iterates over a tetrahedral grid, querying edge intersections, reducing complexity via optional modulo-2 logic, constructing boundary curves to resolve ambiguities, and merging local triangle soups into a global result.

**Inputs and Outputs:**
- **Input:** A tetrahedral grid (accessed via `TetRange` iterator), a scalar field or SDF, an isovalue, and options (`opts`) including `mod2` (boolean), `greedy` (boolean for surface construction strategy), and `mergeEPS` (numerical merge epsilon).
- **Output:** A global `TriangleSoup` containing the extracted isosurface.

**Data Structures:**
- `EdgeInts`: A structure storing intersection counts for the 6 edges of a tetrahedron. Edges are indexed by vertex pairs (0,1), (0,2), (0,3), (1,2), (1,3), (2,3).
- `CombFace`: Represents a curve segment in the combinatorial boundary representation.
- `TriangleSoup`: A container for triangles, supporting combinatorial or numerical merging.

**Steps:**
1. **Iterate:** Loop through each tetrahedron in the grid using a `TetRange` iterator.
2. **Query Intersections:** For the current tetrahedron, query the edge intersection handler for all 6 edges. Request t-values (intersection positions) and normals (if required).
3. **Compute Counts:** Calculate the number of intersections per edge from the returned t-values. Store these in an `EdgeInts` object.
4. **Mod2 Reduction (Optional):** If `opts.mod2` is true, apply modulo-2 reduction to the intersection data. This simplifies the topology by considering intersections modulo 2. If the reduction function returns true, skip processing this tetrahedron.
5. **Construct Boundary Curves:** Call `boundary_comb_curves` with the tetrahedron indices, intersection counts, and edge occupation flags. This function resolves topological ambiguities and yields three types of curves:
   - `open_curves`: Curves that do not form closed loops.
   - `scoop_curves`: Curves associated with scoop configurations.
   - `normal_curves`: Curves associated with normal configurations.
6. **Track Statistics (Optional):**
   - Increment a counter for "non-even" tetrahedra if `open_curves` is not empty.
   - Increment a counter for "non-normal" tetrahedra if `open_curves` is empty AND `scoop_curves` is empty.
7. **Construct Local Surface:** Create a local `TriangleSoup` for the current tetrahedron. Use the `greedy` strategy if `opts.greedy` is true, otherwise use the standard surface construction method.
8. **Merge:** Merge the local `TriangleSoup` into the global result soup.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def subgrid_pipeline(tet_grid, scalar_field, isovalue, opts):
    global_soup = TriangleSoup()
    non_even_count = 0
    non_normal_count = 0

    for tet in tet_grid:
        # Step 2: Query edge intersections
        edge_ints = EdgeInts()
        for edge_idx in range(6):
            t_val, normal = query_edge_intersection(tet, edge_idx, isovalue, scalar_field, opts)
            if t_val is not None:
                edge_ints.set(edge_idx, 1) # Mark intersection
            else:
                edge_ints.set(edge_idx, 0)

        # Step 4: Mod2 Reduction
        if opts.mod2:
            if convert_mod2_intersections(edge_ints):
                continue # Skip this tetrahedron

        # Step 5: Construct boundary curves
        open_curves, scoop_curves, normal_curves = boundary_comb_curves(
            tet.indices, edge_ints, opts
        )

        # Step 6: Track statistics
        if len(open_curves) > 0:
            non_even_count += 1
        if len(open_curves) == 0 and len(scoop_curves) == 0:
            non_normal_count += 1

        # Step 7: Construct local surface
        local_soup = TriangleSoup()
        if opts.greedy:
            construct_greedy_surface(local_soup, tet, edge_ints, open_curves, scoop_curves, normal_curves)
        else:
            construct_standard_surface(local_soup, tet, edge_ints, open_curves, scoop_curves, normal_curves)

        # Step 8: Merge
        global_soup.merge(local_soup, opts.mergeEPS)

    return global_soup, non_even_count, non_normal_count
```

**Parameters:**
- `opts.mod2`: Boolean. If true, applies modulo-2 reduction to intersection data.
- `opts.greedy`: Boolean. If true, uses the greedy surface construction method for optimized performance.
- `opts.mergeEPS`: Float. Positional merge epsilon. If provided, switches from combinatorial merge to numerical merge.

**Edge Cases and Pitfalls:**
- **Mod2 Reduction:** The function `convert_mod2_intersections` must be correctly implemented. It was moved to `query/mod2_reduction.h` in the reference implementation [S1, unofficial].
- **Boundary Curves:** The `boundary_comb_curves` function must correctly handle open, scoop, and normal curves to ensure watertightness. Failure to resolve these correctly leads to non-watertight meshes.
- **Merge Strategy:** The choice between combinatorial and numerical merge (controlled by `mergeEPS`) affects the final mesh quality and connectivity.

## Subtopics

### Mod2 Reduction
A preprocessing step applied to intersection data to reduce the complexity of the subgrid construction by considering intersections modulo 2 [S1, unofficial]. It is called within the dual subgrid pipeline if `opts.mod2` is true. If it returns true, the current tetrahedron is skipped. The function `convert_mod2_intersections` was moved to `query/mod2_reduction.h` [S1, unofficial].

## Operating characteristics
The method processes each tetrahedron independently, allowing for potential parallelization. The complexity is linear with respect to the number of tetrahedra, $O(N)$, where $N$ is the number of tetrahedra. The construction of boundary curves and local triangle soups adds constant overhead per tetrahedron. The final merge step depends on the merge strategy (combinatorial or numerical) and the number of triangles. The method ensures watertightness, which is a significant advantage over standard Marching Tetrahedrons in ambiguous cases. However, the explicit handling of boundary curves may increase computational cost compared to simpler methods.

## Background
The Subgrid Pipeline is part of the `subgrid-marching` project by H. Baktash [S1, unofficial]. It is designed to address the topological ambiguities inherent in the Marching Tetrahedrons algorithm by explicitly resolving open, scoop, and normal boundary curves.

## Sources
[S1] hbaktash/subgrid-marching, https://github.com/hbaktash/subgrid-marching, fetched 2026-10-27 (unofficial)
