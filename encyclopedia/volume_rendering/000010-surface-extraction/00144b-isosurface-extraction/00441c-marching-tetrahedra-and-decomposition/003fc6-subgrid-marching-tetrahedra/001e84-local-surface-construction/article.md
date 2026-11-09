# Local Surface Construction


## Overview
Local Surface Construction is the core geometric engine of Subgrid Marching Tetrahedra, responsible for generating a local, intersection-free triangle soup from a single tetrahedron's edge intersections and curve topology [S1, unofficial]. It solves the problem of ambiguous surface configurations within a tetrahedron by classifying intersection curves into open, scoop, and normal types, ensuring the resulting mesh is globally embeddable without self-intersections [S1, unofficial]. It differs from [Marching Tetrahedrons](3720-marching-tetrahedrons.md) by explicitly handling non-normal curves via smeared polygons and recursive subdivision logic provided by its child [Primal Reconstruction Algorithm](10073-primal-reconstruction-algorithm.md), and from [Dual Subgrid Marching Tetrahedra](10072-dual-subgrid-marching-tetrahedra.md) by operating on the primal grid structure [S1, unofficial].

## How it works
**Inputs:**
- Tetrahedron positions and indices.
- Edge intersection t-values (normalized distances along edges).
- Curve type classification (open, scoop, normal) for each tetrahedron.
- Configuration options: `opts.scoop_mid_vertices` (controls mid-vertex placement), `opts.scoop_bulge` (controls bulging parameter).

**Outputs:**
- A `TriangleSoup` containing local triangles for the tetrahedron.

**Data Structures:**
- `TriangleSoup`: A container for triangles, with a merge flag `COMB_MERGE` to handle shared vertices.
- `local_counts`: Array of 6 integers, counting intersections per local edge.
- `local_ts`: Array of 6 vectors of doubles, storing intersection t-values per local edge.

**Steps:**
1. **Global Count Aggregation:** Compute global intersection counts for all edges in the grid using `build_global_counts()`. This ensures consistency across shared edges between adjacent tetrahedra [S1, unofficial].
2. **Global T-Value Generation:** Generate global t-values for all edges using `build_global_ts(global_counts, rng_seed, use_random_ts)`. If deterministic t-values are not used, random seeds ensure consistent but non-deterministic placement in ambiguous cases [S1, unofficial].
3. **Local Mapping:** For each of the 4 tetrahedra:
   - Map global edges to the local 6-edge frame using `ALL_TET_PAIRS`.
   - Retrieve `local_counts` and `local_ts` for each local edge `ei` by looking up the global edge key `ge = make_edge(tet.idx[li], tet.idx[lj])` [S1, unofficial].
4. **Curve Classification:** Classify the intersection curves within the tetrahedron into three types:
   - **Open Curves:** Standard intersections.
   - **Scoop Curves:** Ambiguous configurations requiring special handling.
   - **Normal Curves:** Standard non-ambiguous configurations.
5. **Local Reconstruction:** Construct the local triangle soup using the classified curves. For scoop curves, apply `opts.scoop_mid_vertices` and `opts.scoop_bulge` to control vertex placement and bulging [S1, unofficial].
6. **Merge:** Combine local triangle soups from all tetrahedra into a global `TriangleSoup`, using `COMB_MERGE` to handle shared vertices [S1, unofficial].

**Pitfalls:**
- If the even-sum condition is not satisfied (tracked via `non_even_tets > 0`), the output mesh may not be orientable and may contain open holes, indicating a non-watertight input or grid edge-intersection issue [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def local_surface_construction(tetrahedra, opts):
    # Step 1: Global Count Aggregation
    global_counts = build_global_counts(tetrahedra)
    
    # Step 2: Global T-Value Generation
    global_ts = build_global_ts(global_counts, rng_seed=0, use_random_ts=False)
    
    TriangleSoup.COMB_MERGE = True
    combined_soup = TriangleSoup()
    
    # Step 3: Process each tetrahedron
    for tet in tetrahedra:
        local_counts = [0] * 6
        local_ts = [[] for _ in range(6)]
        
        for ei in range(6):
            li, lj = ALL_TET_PAIRS[ei]
            ge = make_edge(tet.idx[li], tet.idx[lj])
            local_counts[ei] = global_counts[ge]
            local_ts[ei] = global_ts[ge]
            
        # Step 4: Curve Classification
        open_curves, scoop_curves, normal_curves = classify_curves(local_counts, local_ts, tet)
        
        # Step 5: Local Reconstruction
        local_soup = reconstruct_ring(local_counts, local_ts, open_curves, scoop_curves, normal_curves, opts)
        
        # Step 6: Merge
        combined_soup.merge(local_soup)
        
    return combined_soup
```

## Subtopics

### [Primal Reconstruction Algorithm](10073-primal-reconstruction-algorithm.md)
Handles the geometric resolution of ambiguous configurations via recursive subdivision and smeared polygons. Choose this when dealing with non-normal curves that would otherwise cause self-intersections.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of tetrahedra, assuming constant-time curve classification and reconstruction per tetrahedron.
- **Memory Complexity:** Linear with respect to the number of tetrahedra and edges, storing global counts and t-values.
- **Strengths:** Guarantees intersection-free output; handles ambiguous configurations robustly.
- **Limitations:** Requires careful handling of the even-sum condition to avoid non-orientable meshes; performance depends on the efficiency of global count aggregation.
- **When to use:** When extracting isosurfaces from tetrahedral grids with ambiguous configurations.
- **When not to use:** When input data is non-watertight or grid edge-intersection issues are present.

## Background
Local Surface Construction is part of the Subgrid Marching Tetrahedra method, developed by H. Baktash et al. [S1, unofficial]. It builds on the principles of marching tetrahedra but introduces specialized handling for ambiguous cases to ensure global embedding.

## Sources
[S1] subgrid-marching, https://github.com/hbaktash/subgrid-marching, fetched 2026-10-27 (unofficial)
