# Topologically Correct Marching Cubes


## Overview
Topologically Correct Marching Cubes (TMC) is an extension to the standard Marching Cubes algorithm that guarantees the output mesh is homeomorphic to the trilinear interpolant of the input scalar field within each voxel [S1, unofficial][S2]. While standard Marching Cubes may produce topologically incorrect artifacts (such as disconnected sheets instead of a tunnel) in ambiguous cases, TMC resolves these by inserting additional vertices, ensuring the resulting mesh is 2-manifold and watertight, provided the isosurface does not intersect the domain boundaries [S1, unofficial][S2]. It builds on [Topological Correctness Guarantees](1796-topological-correctness-guarantees.md) by providing a specific implementation for the trilinear interpolant.

## How it works
**Inputs:**
- A voxel grid domain defined by vertex positions and scalar values.
- An isovalue scalar.
- A flag `use_topologically_correct_marching_cubes` set to true [S2].

**Outputs:**
- A set of output points (vertices) and polygons (triangles) representing the isosurface [S2].

**Data Structures:**
- The algorithm processes each voxel cell independently.
- For each cell, it evaluates the scalar values at the 8 vertices.
- If the cell configuration is ambiguous (e.g., Case 13 in the standard lookup table), TMC deviates from the standard 15-case lookup.

**Steps:**
1. **Standard Evaluation:** For each voxel, determine the sign of the scalar value at each of the 8 vertices relative to the isovalue.
2. **Ambiguity Detection:** Identify if the vertex configuration corresponds to an ambiguous case where the standard linear interpolation assumption leads to topological errors (e.g., a tunnel passing through the cell) [S2].
3. **Vertex Insertion:** If an ambiguous case is detected, insert additional vertices within the cell to resolve the topology [S1, unofficial][S2]. The sources indicate these vertices can be placed at the center of the cell or optimized using Quadric Error Metrics (QEM) for Dual Contouring [S1, unofficial].
4. **Triangulation:** Generate triangles connecting the original edge intersections and the newly inserted vertices to form a mesh homeomorphic to the trilinear interpolant [S1, unofficial][S2].
5. **Assembly:** Combine the triangles from all cells into the final polygon mesh [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def topologically_correct_marching_cubes(domain, isovalue):
    points = []
    polygons = []
    
    for cell in domain.cells:
        # Get scalar values at the 8 vertices of the cell
        values = [domain.scalar_value(v) for v in cell.vertices]
        
        # Determine signs relative to isovalue
        signs = [1 if v > isovalue else -1 for v in values]
        
        # Check for ambiguous cases that require topological correction
        # Note: The specific logic for detecting ambiguity and inserting vertices
        # is encapsulated in the CGAL implementation [S2].
        if is_ambiguous_case(signs):
            # Insert additional vertices to resolve topology (e.g., tunnel)
            # Vertices may be placed at cell center or optimized via QEM [S1, unofficial]
            new_vertices = insert_topological_vertices(cell, values, isovalue)
            
            # Generate triangles for the corrected topology
            triangles = generate_corrected_triangles(cell, new_vertices)
        else:
            # Use standard Marching Cubes lookup for non-ambiguous cases
            triangles = standard_marching_cubes_triangles(cell, values, isovalue)
            
        # Add new points and polygons to the output
        points.extend(new_vertices)
        polygons.extend(triangles)
        
    return points, polygons

def is_ambiguous_case(signs):
    # Returns true for configurations like Case 13 where standard MC fails
    # to represent the trilinear interpolant correctly [S2]
    # Implementation details depend on the specific ambiguity detection logic
    # defined in the source paper [S1, unofficial] or CGAL library [S2].
    pass

def insert_topological_vertices(cell, values, isovalue):
    # Inserts vertices to resolve complex features like tunnels [S1, unofficial][S2]
    # Can place at center or optimize using QEM [S1, unofficial]
    pass

def generate_corrected_triangles(cell, new_vertices):
    # Generates triangles homeomorphic to the trilinear interpolant [S1, unofficial]
    pass
```

**Edge Cases and Pitfalls:**
- **Domain Boundaries:** The guarantee of being 2-manifold and watertight holds only if the isosurface does not intersect the domain boundaries [S1, unofficial][S2].
- **Sharp Features:** TMC does not guarantee the recovery of sharp features present in the input value field [S2].
- **Complexity:** The insertion of additional vertices increases the vertex count and computational cost compared to standard MC [S1, unofficial][S2].

## Subtopics

### [Topology-aware mesh quality improvement for Marching Cubes 33](1617-topology-aware-mesh-quality-improvement-for-marching-cubes-3.md)
Improves aspect ratio of MC33 triangulations while preventing topological errors. Unlike TMC, it specifically targets MC33 and uses a verification step to prevent topological errors, whereas TMC is a general extension to standard MC.

## Operating characteristics
- **Time Complexity:** Higher than standard Marching Cubes due to the detection of ambiguous cases and the insertion of additional vertices [S1, unofficial][S2].
- **Memory Complexity:** Increased memory usage due to the potential addition of new vertices per cell [S1, unofficial][S2].
- **Accuracy:** Accurately represents small complex features (e.g., tunnels) that standard MC might miss [S2].
- **Limitations:** Does not guarantee recovery of sharp features [S2]. Requires the isosurface to not intersect domain boundaries for manifold guarantees [S1, unofficial][S2].
- **When to Use:** When topological correctness is critical and the scalar field is smoothly interpolated (trilinear), especially for complex internal structures like tunnels [S1, unofficial][S2].
- **When Not to Use:** When sharp feature preservation is required or when performance is paramount and topological errors are acceptable [S2].

## Background
Topologically Correct Marching Cubes was introduced by Roberto Grosso in the paper "Construction of topologically correct and manifold isosurfaces" published in *Computer Graphics Forum*, volume 35, pages 187–196, by Wiley Online Library in 2016 [S1, unofficial]. It is implemented in the CGAL 3D Isosurfacing package, available since CGAL 6.1 (released fall 2025) [S1, unofficial][S2].

## Sources
[S1] New in CGAL: Isosurfacing, https://www.cgal.org/2025/10/10/Isosurfacing/, fetched 2026-10-28 (unofficial)
[S2] 3D Isosurfacing, https://doc.cgal.org/latest/Isosurfacing_3/index.html, fetched 2026-10-29
