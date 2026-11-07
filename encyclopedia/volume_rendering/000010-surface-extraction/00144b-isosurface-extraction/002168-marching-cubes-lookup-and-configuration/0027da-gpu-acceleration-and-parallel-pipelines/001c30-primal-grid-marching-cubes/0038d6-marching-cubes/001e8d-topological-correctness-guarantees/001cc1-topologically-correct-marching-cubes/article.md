# Topologically Correct Marching Cubes


## Overview
Topologically Correct Marching Cubes (TMC) is an extension to the standard Marching Cubes algorithm that guarantees the output mesh is homeomorphic to the trilinear interpolant of the input scalar field within each cell [S1, unofficial][S2]. While standard Marching Cubes may produce topologically incorrect results (such as disconnected sheets instead of a tunnel) in ambiguous configurations, TMC resolves these ambiguities by inserting additional vertices, ensuring the mesh accurately represents small complex features [S1, unofficial][S2]. It builds on [Topological Correctness Guarantees](1796-topological-correctness-guarantees.md) by using geometric insertion to split cells, differing from [Asymptotic Decider](790-asymptotic-decider.md) which uses continuous field analysis, and [Topologically Defined Isosurfaces](1428-topologically-defined-isosurfaces.md) which operates on discrete connectivity [S1, unofficial][S2].

## How it works
**Inputs:**
*   A domain (space partition) defining the grid structure [S2].
*   A scalar value field associated with the grid vertices [S2].
*   An isovalue scalar parameter [S2].

**Outputs:**
*   A set of points (vertices) representing the isosurface [S2].
*   A set of polygons (triangles) defining the mesh connectivity [S2].

**Data Structures:**
*   The implementation relies on a domain object that encapsulates the space partition, value field, and potentially gradient field [S2].
*   The output is stored in `PointRange` and `PolygonRange` containers [S2].

**Method:**
1.  **Cell Processing:** Iterate over each cell in the domain [S2].
2.  **Ambiguity Detection:** Identify cell configurations where the standard Marching Cubes lookup table yields ambiguous topologies (e.g., cases where the trilinear interpolant forms a tunnel or complex internal structure) [S1, unofficial][S2].
3.  **Vertex Insertion:** For ambiguous cells, insert additional vertices within the cell to resolve the topology [S1, unofficial][S2].
    *   The sources indicate vertices can be placed at the center of the cell or optimized using Quadric Error Metrics (QEM) for Dual Contouring [S1, unofficial].
    *   This insertion splits the cell into sub-cells or modifies the triangulation to ensure the resulting mesh is homeomorphic to the trilinear interpolant [S1, unofficial][S2].
4.  **Triangulation:** Generate triangles based on the modified vertex set, ensuring consistent connectivity across shared faces [S1, unofficial][S2].
5.  **Manifold Guarantee:** The resulting mesh is guaranteed to be 2-manifold and watertight, provided the isosurface does not intersect the domain boundaries [S1, unofficial][S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def topologically_correct_marching_cubes(domain, isovalue):
    """
    Executes Topologically Correct Marching Cubes.
    
    Args:
        domain: Object containing space partition, value field, and gradient field.
        isovalue: Scalar threshold for isosurface extraction.
        
    Returns:
        points: List of 3D points forming the mesh vertices.
        polygons: List of triangle indices forming the mesh faces.
    """
    points = []
    polygons = []
    
    # Iterate over all cells in the domain
    for cell in domain.cells():
        # Get vertex values for the current cell
        vertex_values = [domain.value(v) for v in cell.vertices()]
        
        # Determine the standard Marching Cubes case index
        case_index = compute_mc_case_index(vertex_values, isovalue)
        
        # Check for topological ambiguity
        if is_ambiguous(case_index, vertex_values):
            # Insert additional vertices to resolve topology
            # Source [S1, unofficial] mentions placement at center or QEM optimization
            # Source [S2] states TMC inserts vertices to resolve tunnels
            new_vertices = insert_vertices_for_topology(cell, vertex_values, isovalue)
            
            # Generate triangles using the new vertex set
            # This step ensures homeomorphism to the trilinear interpolant
            triangles = generate_triangles_ambiguous(cell, new_vertices, isovalue)
        else:
            # Standard Marching Cubes triangulation for non-ambiguous cases
            triangles = generate_triangles_standard(cell, vertex_values, isovalue)
            
        # Add new points to the global list
        for v in triangles.vertices:
            if v not in points:
                points.append(v)
                
        # Add triangles to the global list
        for tri in triangles:
            polygons.append([points.index(v) for v in tri.vertices])
            
    return points, polygons

def is_ambiguous(case_index, vertex_values):
    """
    Determines if a cell configuration is topologically ambiguous.
    Standard MC cases 13 and 14 are often ambiguous.
    [G] Implementation detail: Check specific case indices and 
    trilinear saddle point conditions if necessary.
    """
    # Simplified check for demonstration
    # Real implementation requires checking the trilinear interpolant's topology
    return case_index in [13, 14] # Example ambiguous cases

def insert_vertices_for_topology(cell, vertex_values, isovalue):
    """
    Inserts vertices to resolve topological ambiguity.
    [S1, unofficial] Placement at center or optimized via QEM.
    [S2] Resolves tunnels within a single cell.
    """
    # [G] Placeholder for specific insertion logic.
    # Common approach: Add a vertex at the cell center.
    center = cell.center()
    return [center]

def generate_triangles_ambiguous(cell, new_vertices, isovalue):
    """
    Generates triangles for ambiguous cells using inserted vertices.
    Ensures the mesh is homeomorphic to the trilinear interpolant.
    """
    # [G] Placeholder for specific triangulation logic.
    # Connect inserted vertices to boundary intersections appropriately.
    return []

def generate_triangles_standard(cell, vertex_values, isovalue):
    """
    Standard Marching Cubes triangulation.
    """
    # [G] Standard MC lookup table application.
    return []
```

**Parameters:**
*   `use_topologically_correct_marching_cubes`: A named parameter that must be set to `true` to enable TMC [S2].

**Edge Cases and Pitfalls:**
*   **Domain Boundaries:** The guarantee of the mesh being 2-manifold and watertight holds only if the isosurface does not intersect the domain boundaries [S1, unofficial][S2].
*   **Sharp Features:** TMC does not guarantee the recovery of sharp features present in the input value field [S2].
*   **Complexity:** TMC accurately represents small complex features (e.g., tunnels) but may introduce additional vertices, potentially increasing mesh complexity compared to standard MC [S1, unofficial][S2].

## Subtopics

### [Topology-aware mesh quality improvement for Marching Cubes 33](1617-topology-aware-mesh-quality-improvement-for-marching-cubes-3.md)
Improves aspect ratio of triangulations while preventing topological errors using a verification step. It combines topological guarantees of Extended Marching Cubes 33 with skinny triangle removal. Unlike Raman and Wenger, it correctly represents the topology of the trilinear interpolant even when $\lambda=0$.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:**
    *   Generates a mesh homeomorphic to the trilinear interpolant of the input value field [S1, unofficial][S2].
    *   Accurately represents small complex features, such as tunnels within a single cell [S1, unofficial][S2].
    *   Guarantees the mesh is 2-manifold and watertight (provided no boundary intersection) [S1, unofficial][S2].
*   **Limitations:**
    *   Does not guarantee recovery of sharp features [S2].
    *   Requires insertion of additional vertices, which may increase mesh complexity [S1, unofficial][S2].
*   **When to use:** When topological correctness and manifold guarantees are critical, especially for complex scalar fields with internal tunnels or ambiguous configurations.
*   **When not to use:** When sharp feature preservation is required, or when computational resources are limited and topological ambiguity is unlikely or acceptable.

## Background
Topologically Correct Marching Cubes was introduced by Roberto Grosso in the paper "Construction of topologically correct and manifold isosurfaces" published in *Computer Graphics Forum*, volume 35, pages 187–196, by Wiley Online Library in 2016 [S1, unofficial][S2]. It is implemented in the CGAL "3D Isosurfacing" package, which became available in CGAL 6.1, released in the fall of 2025 [S1, unofficial][S2].

## Sources
[S1] New in CGAL: Isosurfacing, https://www.cgal.org/2025/10/10/Isosurfacing/, fetched 2026-10-28 (unofficial)
[S2] 3D Isosurfacing, https://doc.cgal.org/latest/Isosurfacing_3/index.html, fetched 2026-10-29
