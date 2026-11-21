# Marching Cubes for EB Generation

Also called: MC33, Marching-Cubes Geometry Generation

## Overview
This method generates Embedded Boundary (EB) geometry on 3D Cartesian grids by resolving a surface using the Marching Cubes 33 (MC33) lookup tables, computing volume/area fractions, centroids, normals, and connectivity [S1, unofficial]. It improves upon the legacy EB generation method by providing a more accurate single-valued representation of the boundary [S1, unofficial]. It is distinguished from standard isosurface extraction by its integration into the AMReX EB framework, specifically handling exact edge intersections via root finding or STL intersection, caching these intersections in `MCFab` structures for downstream aperture construction, and enforcing single-valued topological constraints [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs**: A geometry source defined as either an implicit function $f(\mathbf{x})$ or a triangulated surface (STL) [S1, unofficial]. A 3D Cartesian grid with cubic cells ($dx = dy = dz$) [S1, unofficial].
*   **Outputs**: A single-valued EB dataset including volume fractions, area fractions, centroids, boundary normals, and connectivity [S1, unofficial]. Resolved MC33 face-connectivity decisions stored in `MCFab` structures [S1, unofficial].

**Data Structures**
*   **MCFab**: A data structure storing resolved MC33 face-connectivity decisions and cached edge intersections (normalized crossing locations) [S1, unofficial]. These cached intersections are shared by downstream geometry calculations including MC vertices, face apertures, edge centroids, and cell moments, including for ambiguous MC33 faces [S1, unofficial].
*   **Level Set**: A nodal level set built from the geometry source. For implicit functions, this is the function value. For STL, a narrow-band level set is used where exact distance values are retained on cut-cell interpolation and vertex-normal stencils, while only inside/outside classification is retained farther from the interface [S1, unofficial].

**Algorithm Steps**

1.  **Grid Validation**: Verify the grid is 3D and cubic ($dx == dy == dz$). Reject if non-cubic [S1, unofficial]. Verify `BuildMultiValuedMultiCut` is not requested [S1, unofficial].

2.  **Level Set Construction**:
    *   If the source is an implicit function, sample $f(\mathbf{x})$ at grid nodes. Do not clamp sample positions to the domain. Snap nodes lying on the surface within roundoff to exact zero [S1, unofficial].
    *   If the source is STL, construct a narrow-band level set. Store exact distance values for nodes near the interface and binary inside/outside classification for distant nodes [S1, unofficial].

3.  **Exact Edge Intersection Computation**:
    *   For every Cartesian edge where the sign of the level set changes (or where the edge intersects the STL geometry), compute the exact intersection location.
    *   **Implicit Functions**: Use Brent root finding on $f(\mathbf{x})$ to find the crossing. If root finding fails, fall back to linear interpolation [S1, unofficial].
    *   **STL**: Intersect the edge with the original triangles using `STLtools::fillMarchingCubesEdgeIntersections` [S1, unofficial].
    *   Store the normalized crossing location (parameter $t \in [0,1]$) in the `MCFab` [S1, unofficial].
    *   If cleanup places the interface exactly on an edge endpoint, use the exact-zero endpoint value [S1, unofficial].

4.  **Case Index Calculation**:
    *   For each voxel, determine the 8-bit case index $C$ based on the signs of the level set values $V_i$ at the 8 vertices $v_i$ relative to the isovalue (typically 0 for implicit functions, or the narrow-band threshold for STL) [S1, unofficial].
    *   Bit $i$ of $C$ is set to 1 if $V(v_i) < 0$ (or $> 0$, depending on convention), else 0. The bit assignment must match the vertex ordering assumed by the LUT generation [S1, unofficial].

5.  **Topology Resolution and Connectivity**:
    *   Use the MC33 lookup tables (`edgeTable`, `triangleTable`) to determine the triangulation for the voxel based on $C$ [S1, unofficial].
    *   For ambiguous cases (46, 47, 98, 99, 210, 213, 214, 215), resolve the topology consistently using the cached edge intersections stored in `MCFab` [S1, unofficial]. This ensures that ambiguous Cartesian faces use the same resolved topology when building face apertures as when generating MC triangles [S1, unofficial].
    *   Store the resolved face-connectivity decisions in `MCFab` [S1, unofficial].

6.  **Geometry Computation**:
    *   Compute MC vertices using the cached exact edge intersections [S1, unofficial].
    *   Compute face apertures, edge centroids, and cell moments using the cached intersections and resolved connectivity [S1, unofficial].
    *   Compute boundary normals and volume/area fractions [S1, unofficial].

7.  **Single-Valued Consistency Checks and Repair**:
    *   Check for single-valued representation violations:
        *   Faces containing more than one resolved fluid aperture.
        *   Cells with invalid MC topology, multiple fluid components, or MC33 "tunnel" tilings where fluid regions are joined only through the cell interior.
        *   Invalid face or cell geometry records.
        *   Cut cells with volume fraction below `eb2.small_volfrac` [S1, unofficial].
    *   **Repair**:
        *   Small cells (below `eb2.small_volfrac`) are removed automatically by moving incident fluid nodes to the interface [S1, unofficial].
        *   Other unsupported geometry (e.g., multi-valued cuts) requires `eb2.cover_multiple_cuts = 1` to cover the geometry, matching the existing behavior for non-single-valued cells [S1, unofficial].
    *   Iteratively remove small cells and unsupported geometry using `eb2.cover_multiple_cuts` and `eb2.maxiter` [S1, unofficial].

8.  **Output**:
    *   Generate the final EB dataset.
    *   Optionally write the MC triangulation as an ASCII STL file if `eb2.mc_stl_file` is specified [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_eb_marching_cubes(grid, geometry_source, params):
    """
    Generate Embedded Boundary geometry using Marching Cubes 33.
    
    Args:
        grid: 3D Cartesian grid with cubic cells (dx == dy == dz).
        geometry_source: Implicit function f(x) or STL triangulation.
        params: Runtime parameters including eb2.small_volfrac, eb2.maxiter, 
                eb2.cover_multiple_cuts, eb2.mc_stl_file.
    
    Returns:
        EB dataset (volume fractions, area fractions, centroids, normals, connectivity).
    """
    
    # 1. Validation
    if grid.dim != 3:
        raise ValueError("Marching Cubes EB generation requires 3D grid")
    if not grid.is_cubic():
        raise ValueError("Marching Cubes EB generation requires cubic cells (dx == dy == dz)")
    
    # 2. Level Set Construction
    level_set = build_nodal_level_set(grid, geometry_source)
    # Snap nodes on surface to exact zero
    level_set = snap_to_zero(level_set, tolerance=roundoff_error)
    
    # 3. Exact Edge Intersections
    mcfab = MCFab()
    for edge in grid.edges:
        if edge.has_sign_change(level_set) or edge.intersects_stl(geometry_source):
            if geometry_source.is_implicit():
                # Brent root finding on implicit function
                t = brent_root_find(edge.start, edge.end, geometry_source.function)
                if t is None:
                    # Fallback to linear interpolation
                    t = linear_interpolate(edge.start, edge.end, level_set)
            else:
                # STL intersection
                t = stltools_fill_marching_cubes_edge_intersections(edge, geometry_source.triangles)
            
            # Store normalized crossing location
            mcfab.set_edge_intersection(edge, t)
    
    # 4. Case Index and Topology Resolution
    for cell in grid.cells:
        # Calculate case index C from level set signs at 8 vertices
        C = compute_case_index(cell.vertices, level_set)
        
        # Resolve topology using MC33 LUTs and cached intersections
        # This handles ambiguous cases consistently
        connectivity = resolve_mc33_topology(C, mcfab.get_cached_intersections(cell.edges))
        
        # Store connectivity in MCFab
        mcfab.set_face_connectivity(cell, connectivity)
    
    # 5. Geometry Computation
    eb_dataset = EBDataset()
    for cell in grid.cells:
        # Use cached intersections for vertices, apertures, centroids, moments
        eb_dataset.compute_cell_moments(cell, mcfab)
        eb_dataset.compute_face_apertures(cell, mcfab)
        eb_dataset.compute_boundary_normals(cell, mcfab)
    
    # 6. Single-Valued Checks and Repair
    max_iter = params.get('eb2.maxiter', 10)
    small_volfrac = params.get('eb2.small_volfrac', 1e-6)
    cover_multi = params.get('eb2.cover_multiple_cuts', 0)
    
    for iteration in range(max_iter):
        violations = check_single_valued_consistency(eb_dataset, mcfab)
        
        if not violations:
            break
            
        for violation in violations:
            if violation.type == 'small_cell':
                if violation.volume_fraction < small_volfrac:
                    # Remove small cell by moving fluid nodes to interface
                    repair_small_cell(violation.cell, mcfab)
            elif violation.type == 'unsupported_geometry':
                if cover_multi:
                    # Cover unsupported geometry
                    cover_unsupported_geometry(violation.cell, mcfab)
                else:
                    raise ValueError("Unsupported geometry detected and eb2.cover_multiple_cuts is 0")
    
    # 7. Output
    if params.get('eb2.mc_stl_file'):
        write_eb_surface_stl(eb_dataset, params['eb2.mc_stl_file'])
        
    return eb_dataset

def build_nodal_level_set(grid, geometry_source):
    """
    Build nodal level set from geometry source.
    For implicit functions, sample f(x) without clamping.
    For STL, build narrow-band level set.
    """
    level_set = zeros_like(grid.nodes)
    if geometry_source.is_implicit():
        for node in grid.nodes:
            level_set[node] = geometry_source.function(node.position)
    else:
        # STL narrow-band level set
        for node in grid.nodes:
            dist = geometry_source.distance(node.position)
            if dist < narrow_band_width:
                level_set[node] = dist
            else:
                level_set[node] = sign(dist) * narrow_band_width
    return level_set

def brent_root_find(start, end, func):
    """
    Find root of func in [start, end] using Brent's method.
    Returns normalized parameter t in [0, 1] or None if failed.
    """
    # Standard Brent's method implementation
    # ...
    pass

def stltools_fill_marching_cubes_edge_intersections(edge, triangles):
    """
    Intersect edge with STL triangles.
    Returns normalized parameter t in [0, 1] or None if no intersection.
    """
    # Ray-triangle intersection for each triangle
    # ...
    pass

def resolve_mc33_topology(case_index, cached_intersections):
    """
    Resolve MC33 topology for a given case index using cached edge intersections.
    Handles ambiguous cases consistently.
    """
    # Use edgeTable and triangleTable from parent article
    # For ambiguous cases, use cached_intersections to determine consistent topology
    # ...
    pass

def check_single_valued_consistency(eb_dataset, mcfab):
    """
    Check for single-valued representation violations.
    Returns list of violations.
    """
    violations = []
    for face in eb_dataset.faces:
        if face.num_fluid_apertures > 1:
            violations.append({'type': 'multi_aperture', 'face': face})
    for cell in eb_dataset.cells:
        if not is_valid_mc_topology(cell):
            violations.append({'type': 'invalid_topology', 'cell': cell})
        if cell.num_fluid_components > 1:
            violations.append({'type': 'multi_component', 'cell': cell})
        if is_tunnel_tiling(cell):
            violations.append({'type': 'tunnel_tiling', 'cell': cell})
        if cell.volume_fraction < eb2.small_volfrac:
            violations.append({'type': 'small_cell', 'cell': cell})
    return violations

def repair_small_cell(cell, mcfab):
    """
    Remove small cell by moving incident fluid nodes to the interface.
    """
    # Move nodes to interface based on cached intersections
    # ...
    pass

def cover_unsupported_geometry(cell, mcfab):
    """
    Cover unsupported geometry by marking cell as solid or fluid.
    """
    # ...
    pass
```

## Operating characteristics
*   **Time Complexity**: Linear in the number of voxels and edges, dominated by the root finding or STL intersection for each sign-changing edge [S1, unofficial].
*   **Memory Complexity**: Proportional to the number of voxels and edges, storing the `MCFab` data structure with cached edge intersections and connectivity decisions [S1, unofficial].
*   **Measured Performance**: The EB produced by Marching Cubes is much better than the existing legacy method [S1, unofficial].
*   **Strengths**: Exact edge intersections avoid inconsistencies from linear interpolation of sampled level-set values [S1, unofficial]. Consistent handling of ambiguous cases prevents cracks or holes [S1, unofficial]. Integration with AMReX EB framework allows for efficient single-valued representation [S1, unofficial].
*   **Limitations**: Only available in 3D on Cartesian grids with cubic cells [S1, unofficial]. Incompatible with `BuildMultiValuedMultiCut` [S1, unofficial]. Requires iterative cleanup for small cells and unsupported geometry [S1, unofficial].
*   **When to Use**: When high-fidelity EB geometry is required on 3D Cartesian grids with cubic cells, and the geometry source is an implicit function or STL [S1, unofficial].
*   **When Not to Use**: On non-cubic grids, in 2D, or when multi-valued cuts are required [S1, unofficial].

## Background
This method was introduced in the AMReX framework via Pull Request #5607 by ankithadas in August 2026 [S1, unofficial]. It extends the Marching Cubes algorithm to EB generation, addressing limitations of the legacy EB construction method [S1, unofficial]. It is used in AMReX for EB generation on 3D Cartesian grids with cubic cells, including tests for box, sphere, gyroid, parser implicit functions, STL geometries (cube, multiple cubes, Stanford bunny), periodic domains, and CNS Sod MarchingCubes_3d test [S1, unofficial].

## Sources
[S1] Marching Cubes for EB Generation, https://github.com/AMReX-Codes/amrex/pull/5607, fetched 2026-10-26 (unofficial)
