# Lookup-table-free voxel meshing via connected wireframes

Also called: Connected wireframe triangulation, Generalized Transvoxel alternative

## Overview
This method extracts an isosurface from a scalar volume by constructing connected components of inside/outside vertices and edge segments, then triangulating the resulting non-planar polygons, thereby eliminating the need for Marching Cubes lookup tables [S1, unofficial]. It differs from sibling methods like [Enhanced Marching Cubes](13017-enhanced-marching-cubes.md) and [Ambiguous Face Interpolant (Red Path)](14165-ambiguous-face-interpolant-red-path.md) by resolving topological ambiguities through global connectivity analysis of wireframes rather than local cell configurations or interpolant adjustments [S1, unofficial].

## How it works
The algorithm operates in four pipeline stages: [A] scalar function evaluation, [B] voxel grid generation, [C] cut point extraction, and [D] mesh triangulation [S1, unofficial].

### Inputs and Outputs
*   **Input:** A scalar field function $f(x, y, z)$ and a grid of evaluation points (voxels).
*   **Output:** A triangulated surface mesh.

### Data Structures
The core intermediate structure is the **weave**, which represents the intersection of the isosurface with the voxel lattice edges.

1.  **Fibers:** For a block of $N \times N \times N$ cells (e.g., $16 \times 16 \times 16$), there are $3 \times (N+1) \times (N+1)$ fibers running parallel to the X, Y, and Z axes through the nodes [S1, unofficial].
2.  **Segments:** Each fiber contains a series of segments marking volume entry and exit points. Crossing points must be consistent across adjacent cells [S1, unofficial].
3.  **Weave Layout:** The weave is stored as a flat list of tuples `(fibre-index, endpoint, normal)` [S1, unofficial].
    *   `fibre-index`: Identifies the specific fiber.
    *   `endpoint`: The 3D coordinate of the segment endpoint (mesh vertex).
    *   `normal`: The surface normal at the endpoint, packed into 3 bytes [S1, unofficial].
    *   This packing results in lower memory usage than the original 3D voxel volume [S1, unofficial].

### Algorithm Steps

1.  **Evaluate Scalar Field:** Compute $f(x, y, z)$ on the space grid (Stage B) [S1, unofficial].
2.  **Resolve Ambiguities:** For any face with ambiguous topology (where vertices alternate inside/outside), test the interiorness function at the center of the quad [S1, unofficial].
    *   If the center is **outside**, use topology configuration (a).
    *   If the center is **inside**, use topology configuration (b).
    *   This prevents long thin channels from fragmenting into disconnected pockets [S1, unofficial].
3.  **Construct Wireframes (Stage C):**
    *   Identify vertices and edge segments that are "inside" the volume (where $f(x,y,z) \le 0$ or similar threshold) [S1, unofficial].
    *   Paint these elements "green" to define connected components of wires with cut ends [S1, unofficial].
    *   Use a `cut face traversing function` to link polyhedron edges (e.g., using a foreright/backleft linking strategy) to ensure consistent crossing points [S1, unofficial].
4.  **Extract Polygon Vertices:**
    *   For each connected wire component, extract the endpoints of the segments [S1, unofficial].
    *   These endpoints form the vertices of a non-planar polygon.
    *   Ensure vertices are ordered correctly along the wire path [S1, unofficial].
    *   Allocate and index these vertices upfront to avoid runtime recalculation of normals [S1, unofficial].
5.  **Triangulate Non-Planar Polygons:**
    *   Triangulate each polygon formed by the wire endpoints [S1, unofficial].
    *   Select diagonal cuts that align with face normals to ensure better folding behavior [S1, unofficial].
    *   For a simple 4-point polygon, choose the diagonal indicated by the normals [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mesh_volume(f, grid_points, threshold=0.0):
    """
    f: scalar function f(x,y,z)
    grid_points: list of 3D coordinates
    threshold: isovalue
    """
    # Stage B: Evaluate
    values = [f(p) for p in grid_points]
    
    # Identify inside/outside status
    inside = [v <= threshold for v in values]
    
    # Stage C: Construct Weave
    # Define fibers (lines along axes)
    fibers = create_fibers(grid_points)
    
    weave_segments = []
    
    for fiber in fibers:
        # Get points along the fiber
        points = fiber.points
        vals = [f(p) for p in points]
        
        # Find segments crossing the threshold
        segments = []
        for i in range(len(points) - 1):
            v1 = vals[i]
            v2 = vals[i+1]
            p1 = points[i]
            p2 = points[i+1]
            
            # Check if segment crosses threshold
            if (v1 <= threshold) != (v2 <= threshold):
                # Interpolate cut point
                t = (threshold - v1) / (v2 - v1)
                cut_point = lerp(p1, p2, t)
                
                # Calculate normal at cut point
                normal = compute_normal(f, cut_point)
                
                segments.append({
                    'fiber_id': fiber.id,
                    'point': cut_point,
                    'normal': normal,
                    'is_inside_start': v1 <= threshold
                })
        
        weave_segments.extend(segments)
        
    # Resolve ambiguities on faces
    # Note: This requires checking face centers as per [S1, unofficial]
    resolve_face_ambiguities(weave_segments, f, grid_points, threshold)
    
    # Group into connected components (wires)
    # This step links segments that share endpoints or are topologically connected
    components = group_connected_components(weave_segments)
    
    # Stage D: Triangulate
    mesh_vertices = []
    mesh_indices = []
    
    for comp in components:
        # Extract ordered endpoints
        polygon_vertices = extract_ordered_endpoints(comp)
        
        # Triangulate non-planar polygon
        # Select diagonals based on normals
        triangles = triangulate_non_planar_polygon(polygon_vertices)
        
        for tri in triangles:
            # Add vertices to global list if not already present
            # Add indices to mesh
            pass
            
    return Mesh(mesh_vertices, mesh_indices)

def resolve_face_ambiguities(segments, f, grid_points, threshold):
    """
    Resolves ambiguous faces by testing the center of the quad.
    """
    # Identify ambiguous faces (alternating inside/outside vertices)
    ambiguous_faces = find_ambiguous_faces(grid_points, threshold)
    
    for face in ambiguous_faces:
        center = compute_face_center(face.vertices)
        val_center = f(center)
        
        if val_center > threshold:
            # Center is outside, use topology (a)
            apply_topology_a(face)
        else:
            # Center is inside, use topology (b)
            apply_topology_b(face)

def compute_normal(f, point):
    """
    Compute normal using central differences [G].
    """
    eps = 1e-5
    dx = f(point + [eps, 0, 0]) - f(point - [eps, 0, 0])
    dy = f(point + [0, eps, 0]) - f(point - [0, eps, 0])
    dz = f(point + [0, 0, eps]) - f(point - [0, 0, eps])
    return normalize([dx, dy, dz])
```

### Edge Cases and Pitfalls
*   **Boundary Ambiguity:** A full voxel meeting a half-voxel diagonally on only one edge creates topological ambiguity requiring specialized handling [S1, unofficial].
*   **Fragmentation:** Without ambiguity resolution (testing the face center), long thin channels may fragment into disconnected pockets [S1, unofficial].
*   **Vertex Ordering:** Incorrect ordering of polygon vertices during extraction leads to invalid triangulation [S1, unofficial].

## Operating characteristics
*   **Memory:** The weave structure (packed as `(fibre-index, endpoint, normal)`) uses less memory than the original 3D voxel volume, as it only encodes the surface [S1, unofficial].
*   **Flexibility:** Eliminates lookup tables, enabling direct support for non-cubic or subdividing lattices without topology redefinition [S1, unofficial].
*   **Editability:** Editing targets Stage C (cut points) rather than Stage B (voxel values), allowing for complex shapes (e.g., tubes around lattice lines) and reducing memory overhead [S1, unofficial].
*   **Complexity:** The cut face traversing function is concise (~10 lines) when using a foreright/backleft linking strategy [S1, unofficial].

## Background
This approach was developed as an alternative to Marching Cubes and Transvoxel, particularly for 2.5D voxel-type surface meshing in 3-axis CNC simulation [S1, unofficial]. The author, goatchurchprime, proposed it in a GitHub issue discussion regarding Zylann's Godot Voxel plugin, suggesting that lookup tables are a "red herring" and that a general-purpose algorithm based on connected wireframes is more robust for subdividing lattices [S1, unofficial].

## Sources
[S1] Generalized alternative implementation to transvoxel, https://github.com/Zylann/godot_voxel/issues/144, fetched 2026-10-28 (unofficial)
