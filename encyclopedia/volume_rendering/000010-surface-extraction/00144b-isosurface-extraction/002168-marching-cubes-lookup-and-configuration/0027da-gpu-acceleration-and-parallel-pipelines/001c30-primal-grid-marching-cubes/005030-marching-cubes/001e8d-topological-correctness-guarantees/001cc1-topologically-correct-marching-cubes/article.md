# Topologically Correct Marching Cubes


## Overview
Topologically Correct Marching Cubes (TMC) is an extension to the standard Marching Cubes algorithm that guarantees the output mesh is homeomorphic to the trilinear interpolant of the input scalar field within each voxel [S1, unofficial][S2]. It solves the ambiguity problem by inserting additional vertices inside cells to resolve complex topological features, such as tunnels or handles, which standard linear interpolation fails to represent correctly [S1, unofficial][S2]. Unlike standard Marching Cubes, which may produce disconnected sheets for a single continuous tunnel, TMC ensures the mesh accurately reflects the topology of the underlying trilinear field [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar field defined on a 3D grid (domain), an isovalue threshold, and a domain definition specifying the grid structure and vertex values [S2].
*   **Output:** A polygonal mesh (typically triangles) representing the isosurface [S2].

**Data Structures:**
The algorithm operates on a voxel grid. For each cube (voxel), it evaluates the scalar values at the 8 vertices. If the standard Marching Cubes lookup table indicates a cut, TMC checks for topological ambiguity. If ambiguity is detected (specifically cases involving saddle points that create tunnels or complex internal structures), the algorithm inserts a new vertex.
*   **Vertex Insertion:** A new vertex is placed at the center of the cell or at an optimized position [S1, unofficial]. The sources mention placement at the center of the cell or optimization using Quadric Error Metrics (QEM) for Dual Contouring, but for TMC specifically, the core mechanism is the insertion of vertices to resolve topology [S1, unofficial].

**Steps:**
1.  Iterate over all voxels in the domain [S2].
2.  For each voxel, determine the intersection case based on the signs of the scalar values at the 8 vertices relative to the isovalue [S2].
3.  If the case is one of the standard 15 ambiguous cases (or specifically those requiring topological correction), analyze the trilinear interpolant within the cell [S1, unofficial][S2].
4.  If the trilinear interpolant contains a saddle point that results in a topology not representable by the standard 14 basic configurations (e.g., a tunnel passing through the cell), insert a new vertex [S1, unofficial][S2].
5.  Connect the new vertex to the existing edge intersection points to form a triangulation that is homeomorphic to the trilinear surface [S1, unofficial][S2].
6.  Assemble the triangles from all voxels into a global mesh [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def topologically_correct_marching_cubes(domain, isovalue):
    points = []
    polygons = []
    
    for voxel in domain.voxels:
        # Get scalar values at the 8 vertices of the voxel
        values = [domain.scalar(v) for v in voxel.vertices]
        
        # Determine the standard Marching Cubes case index
        case_index = compute_mc_case_index(values, isovalue)
        
        # Check if the case requires topological correction
        # Standard MC handles 14 cases. Cases 15-23 are ambiguous.
        # TMC specifically handles cases where the trilinear surface has a saddle point
        # that creates a tunnel or complex topology not captured by linear edges.
        if case_index in AMBIGUOUS_CASES:
            # Analyze the trilinear interpolant to detect saddle points
            # This involves checking the discriminant or critical points of the trilinear function
            has_tunnel = detect_trilinear_tunnel(values, isovalue)
            
            if has_tunnel:
                # Insert a new vertex at the center of the cell
                # [G] The exact position might be the geometric center or an optimized point
                center = voxel.center()
                new_vertex_index = points.append(center)
                
                # Generate triangles connecting the new vertex to the edge intersections
                # The edge intersections are computed as in standard MC
                edge_intersections = compute_edge_intersections(values, isovalue, voxel)
                
                # Connect new_vertex_index to the appropriate edge intersections
                # to form a topology-correct triangulation
                triangles = generate_tunnel_triangles(new_vertex_index, edge_intersections)
                polygons.extend(triangles)
            else:
                # Fall back to standard Marching Cubes for non-tunnel ambiguous cases
                # or cases where the topology is simple
                triangles = standard_mc_triangulation(values, isovalue, voxel)
                polygons.extend(triangles)
        else:
            # Standard non-ambiguous case
            triangles = standard_mc_triangulation(values, isovalue, voxel)
            polygons.extend(triangles)
            
    return points, polygons

def detect_trilinear_tunnel(values, isovalue):
    # [G] This function would analyze the trilinear interpolant
    # to determine if it contains a saddle point that creates a tunnel.
    # This typically involves solving for critical points of the trilinear function
    # and checking their values relative to the isovalue.
    # Not fully specified in sources, but implied by "analyze the trilinear interpolant".
    return False # Placeholder for actual topological analysis
```

**Parameters:**
*   `use_topologically_correct_marching_cubes`: A boolean flag. Set to `true` to enable TMC [S2].

**Edge Cases and Pitfalls:**
*   **Boundary Intersection:** The guarantee of the mesh being 2-manifold and watertight holds only if the isosurface does not intersect the domain boundaries [S1, unofficial][S2].
*   **Sharp Features:** TMC does not guarantee the recovery of sharp features present in the input field [S2].
*   **Complexity:** The insertion of vertices increases the vertex count and complexity of the mesh compared to standard MC [S2].

## Subtopics

### [Topology-aware mesh quality improvement for Marching Cubes 33](1617-topology-aware-mesh-quality-improvement-for-marching-cubes-3.md)
Improves mesh quality while maintaining topological correctness using Extended Marching Cubes 33 and skinny triangle removal.

## Operating characteristics
*   **Time Complexity:** Higher than standard Marching Cubes due to the additional topological analysis and vertex insertion steps [S2].
*   **Memory Complexity:** Higher than standard Marching Cubes due to the potential insertion of additional vertices [S2].
*   **Strengths:** Guarantees topological correctness of the trilinear interpolant; accurately represents small complex features like tunnels [S1, unofficial][S2].
*   **Limitations:** Does not guarantee sharp feature recovery; increased computational cost [S2].
*   **When to use:** When topological accuracy is critical, especially for complex internal structures like tunnels or handles [S1, unofficial][S2].
*   **When not to use:** When sharp feature preservation is required, or when computational resources are limited [S2].

## Background
Topologically Correct Marching Cubes was introduced by Roberto Grosso in the paper "Construction of topologically correct and manifold isosurfaces" published in *Computer Graphics Forum*, volume 35, pages 187–196, by Wiley Online Library in 2016 [S1, unofficial]. It is implemented in the CGAL 3D Isosurfacing package, available since CGAL 6.1 (released in the fall of 2025) [S1, unofficial].

## Sources
[S1] New in CGAL: Isosurfacing, https://www.cgal.org/2025/10/10/Isosurfacing/, fetched 2026-10-28 (unofficial)
[S2] 3D Isosurfacing, https://doc.cgal.org/latest/Isosurfacing_3/index.html, fetched 2026-10-29
