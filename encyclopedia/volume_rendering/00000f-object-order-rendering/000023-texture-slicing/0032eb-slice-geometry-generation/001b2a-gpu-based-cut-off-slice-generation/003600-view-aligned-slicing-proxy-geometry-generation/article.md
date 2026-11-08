# View-Aligned Slicing (Proxy Geometry Generation)

Also called: Bounding Box Plane Intersection, Cube-Plane Intersection Slicing, Geometric Slice Casting, Planar Slice Volume Rendering, Slice-Base Volume Rendering, Volume Slicing

## Overview
View-Aligned Slicing is a volume rendering technique that approximates a volumetric dataset by intersecting its axis-aligned bounding box with a series of parallel planes aligned with the view direction, generating a polygon mesh for each slice [S1, S2, unofficial]. It differs from standard texture slicing by explicitly constructing geometry (proxy meshes) for each cross-section, allowing the use of standard rasterization pipelines for rendering the volume as a collection of triangles rather than sampling a 3D texture [S1, S2, unofficial]. This method builds on [GPU-based Cut-off Slice Generation](948-gpu-based-cut-off-slice-generation.md) by providing the specific geometric construction logic for view-aligned planes.

## How it works
### Inputs and Outputs
**Inputs:**
*   `cameraPos`: Camera position vector.
*   `target`: Camera target (look-at point) vector.
*   `cubeVertices`: Array of 8 vertices defining the volume's bounding box. In the reference implementation, these are unit cube vertices transformed by a 45-degree rotation around the X-axis [S2, unofficial].
*   `numberOfSlices`: Integer count of slices to generate (default 10) [S2, unofficial].
*   `sliceScale`: Scaling factor for slice spacing [S1, unofficial].
*   `EPSILON`: Small float constant to prevent missed intersections due to precision errors [S2, unofficial].

**Outputs:**
*   A list of meshes (or vertex/index buffers), one per slice, containing triangulated polygons representing the intersection of the slice plane with the bounding box.

### Data Structures
*   `cube_vertices[8]`: `glm::vec3` array. The 8 vertices of the bounding box [S2, unofficial].
*   `cube_edges[12]`: Pre-defined indices or vectors representing the 12 edges of the cube. Each edge is defined by a start vertex and a direction vector [S2, unofficial].
*   `lambda[12]`: `float` array. Stores the intersection parameter $\lambda$ for each of the 12 edges for the current slice plane [S2, unofficial].
*   `lambda_inc[12]`: `float` array. Stores the increment of $\lambda$ per slice step for each edge [S2, unofficial].
*   `intersection[6]`: `glm::vec3` array. Temporary storage for up to 6 intersection points found on the cube edges for a single slice [S2, unofficial].
*   `vTextureSlices`: Container for the final generated geometry (vertices and indices) [S2, unofficial].

### Algorithm Steps

1.  **Compute View Direction:**
    Calculate the view direction vector $V$ by subtracting the target from the camera position and normalizing:
    $$ V = \text{normalize}(\text{cameraPos} - \text{target}) $$
    This vector defines the normal for all slice planes [S1, S2, unofficial].

2.  **Determine Slice Extents:**
    Compute the dot product of $V$ with all 8 `cube_vertices` to find the minimum ($min\_z$) and maximum ($max\_z$) distances along the view axis [S2, unofficial].
    Expand these bounds by `EPSILON` to ensure full coverage:
    $$ min\_z \leftarrow min\_z - \text{EPSILON} $$
    $$ max\_z \leftarrow max\_z + \text{EPSILON} $$
    Identify the bounding box vertex closest to the camera (minimum dot product) to determine the starting position [S1, unofficial].

3.  **Calculate Initial Plane Offset and Spacing:**
    Calculate the initial plane offset $P_{offset}$ by projecting the closest vertex onto the viewing direction relative to the target [S1, unofficial].
    Calculate the slice spacing $\Delta Z$:
    $$ \Delta Z = \frac{max\_z - min\_z}{\text{numberOfSlices}} $$
    The plane increment $P_{inc}$ is derived from $\Delta Z$ scaled by `sliceScale` [S1, unofficial].

4.  **Initialize Intersection Parameters:**
    Invert the plane offset and initialize the intersect plane with normal $V$ and offset [S1, unofficial].
    For each of the 12 cube edges $i$:
    *   Compute denominator: $d_i = \text{dot}(\text{edgeDir}[i], V)$ [S2, unofficial].
    *   Check for parallel edges: If $1.0 + d_i \neq 1.0$ (i.e., $d_i$ is not effectively zero), compute:
        $$ \lambda_{inc}[i] = \frac{P_{inc}}{d_i} $$
        $$ \lambda[i] = \frac{P_{offset} - \text{dot}(\text{edgeStart}[i], V)}{d_i} $$
    *   Otherwise (edge parallel to view), set $\lambda[i] = -1.0$ and $\lambda_{inc}[i] = 0.0$ [S2, unofficial].

5.  **Generate Slices:**
    Calculate the total number of slices to ensure coverage:
    $$ \text{numSlices} = \text{round}\left(\frac{P_{offset}}{P_{inc}}\right) \times 2 $$
    [S1, unofficial]

    Iterate through slice indices $i$ from $\text{numSlices} - 1$ down to $0$ [S2, unofficial]:
    *   Initialize an empty list for intersection points.
    *   For each edge $e$ from 0 to 11:
        *   Compute the current intersection parameter:
            $$ dL[e] = \lambda[e] + i \times \lambda_{inc}[e] $$
        *   If $0.0 \le dL[e] < 1.0$:
            *   Compute intersection point: $P = \text{edgeStart}[e] + dL[e] \times \text{edgeDir}[e]$.
            *   Add $P$ to the intersection list [S2, unofficial].
    *   **Pitfall:** If the number of intersection points is $\le 2$, skip this slice (indicates co-planar or no intersection) [S1, unofficial].
    *   **Triangulation:** Use fan triangulation on the collected points (up to 6). The fixed index pattern for points $0..k$ is:
        *   3 points: $(0, 2, 1)$
        *   4 points: $(0, 2, 1), (0, 3, 2)$
        *   5 points: $(0, 2, 1), (0, 3, 2), (0, 4, 3)$
        *   6 points: $(0, 2, 1), (0, 3, 2), (0, 4, 3), (0, 5, 4)$
        [S1, S2, unofficial]
    *   Append the resulting vertices and indices to `vTextureSlices`.
    *   Advance the plane offset: $P_{offset} \leftarrow P_{offset} + P_{inc}$ [S1, unofficial].

6.  **Post-Processing:**
    *   Merge duplicate vertices in the generated geometry to clean up the point cloud [S1, unofficial].
    *   Compute bounding spheres and face normals for the triangulated geometry [S1, unofficial].
    *   Create the final mesh. Set render order to $\text{numSlices} - i$ to ensure correct back-to-front drawing (painter's algorithm) [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def view_aligned_slicing(camera_pos, target, cube_vertices, num_slices_default=10, slice_scale=1.0, epsilon=1e-6):
    """
    Generates proxy geometry for volume rendering by intersecting view-aligned planes with a bounding box.
    
    Args:
        camera_pos: vec3, camera position
        target: vec3, camera look-at target
        cube_vertices: list of 8 vec3, bounding box vertices
        num_slices_default: int, default number of slices
        slice_scale: float, scaling factor for slice spacing
        epsilon: float, small constant for precision safety
        
    Returns:
        list of Mesh objects, each containing vertices and indices for a slice
    """
    # 1. Compute View Direction
    view_dir = normalize(camera_pos - target)
    
    # 2. Determine Slice Extents
    # Project all cube vertices onto view_dir
    projections = [dot(v, view_dir) for v in cube_vertices]
    min_z = min(projections) - epsilon
    max_z = max(projections) + epsilon
    
    # Find closest vertex index to camera (min projection)
    closest_idx = argmin(projections)
    closest_vertex = cube_vertices[closest_idx]
    
    # 3. Calculate Initial Plane Offset and Spacing
    # Project closest vertex onto view direction relative to target
    # Note: Source [S1, unofficial] says "projecting the closest vertex onto the viewing direction, 
    # subtracting the target, and taking the length". 
    # Geometrically, this is the distance along the view axis.
    plane_offset = dot(closest_vertex - target, view_dir)
    
    delta_z = (max_z - min_z) / num_slices_default
    plane_inc = delta_z * slice_scale
    
    # 4. Initialize Intersection Parameters for 12 Edges
    # Define the 12 edges of the cube. 
    # Assuming cube_vertices are ordered such that edges connect specific indices.
    # Standard cube edge indices (0-7):
    # 0-1, 1-3, 3-2, 2-0 (bottom), 4-5, 5-7, 7-6, 6-4 (top), 0-4, 1-5, 2-6, 3-7 (vertical)
    # This mapping depends on the specific vertex ordering in cube_vertices.
    # We assume a helper function get_cube_edges(cube_vertices) returns list of (start, direction) tuples.
    edges = get_cube_edges(cube_vertices)
    
    lambda_vals = [-1.0] * 12
    lambda_incs = [0.0] * 12
    
    for i, (edge_start, edge_dir) in enumerate(edges):
        denom = dot(edge_dir, view_dir)
        # Check if denom is effectively zero
        if 1.0 + denom != 1.0:
            lambda_incs[i] = plane_inc / denom
            lambda_vals[i] = (plane_offset - dot(edge_start, view_dir)) / denom
        else:
            lambda_vals[i] = -1.0
            lambda_incs[i] = 0.0
            
    # 5. Generate Slices
    # Calculate total slices needed to cover the volume
    # Source [S1, unofficial]: numSlices = Math.round(planeOffset / planeInc) * 2
    # This formula seems to double the count to ensure full coverage from min to max.
    # However, the loop in [S2, unofficial] iterates from num_slices-1 down to 0.
    # We will use the calculated numSlices from [S1, unofficial] for the loop count.
    num_slices = round(plane_offset / plane_inc) * 2
    
    meshes = []
    
    for i in range(num_slices - 1, -1, -1):
        intersection_points = []
        
        # Find intersections for this slice
        for e in range(12):
            dL = lambda_vals[e] + i * lambda_incs[e]
            
            if 0.0 <= dL < 1.0:
                # Compute intersection point
                p = edge_start[e] + dL * edge_dir[e]
                intersection_points.append(p)
                
        # Pitfall: Skip if <= 2 points
        if len(intersection_points) <= 2:
            continue
            
        # Triangulate using fan
        vertices = []
        indices = []
        
        # Fan triangulation indices based on source [S1, unofficial]
        # Pattern: (0,2,1), (0,3,2), (0,4,3)...
        # This implies a specific winding order.
        if len(intersection_points) == 3:
            indices = [0, 2, 1]
        elif len(intersection_points) == 4:
            indices = [0, 2, 1, 0, 3, 2]
        elif len(intersection_points) == 5:
            indices = [0, 2, 1, 0, 3, 2, 0, 4, 3]
        elif len(intersection_points) == 6:
            indices = [0, 2, 1, 0, 3, 2, 0, 4, 3, 0, 5, 4]
        else:
            # Fallback for unexpected counts (e.g., if sorting was skipped)
            # Standard fan from vertex 0
            for k in range(1, len(intersection_points) - 1):
                indices.extend([0, k, k + 1])
                
        # Add vertices
        for p in intersection_points:
            vertices.append(p)
            
        # Create Mesh
        mesh = Mesh(vertices, indices)
        
        # Post-processing: Merge duplicates, compute normals
        mesh.merge_duplicate_vertices()
        mesh.compute_face_normals()
        mesh.compute_bounding_sphere()
        
        # Set render order
        mesh.render_order = num_slices - i
        
        meshes.append(mesh)
        
    return meshes

def get_cube_edges(cube_vertices):
    """
    Returns list of (start_vertex, direction_vector) for the 12 edges of a cube.
    Assumes standard cube vertex ordering:
    0: (-1,-1,-1), 1: (1,-1,-1), 2: (-1,1,-1), 3: (1,1,-1)
    4: (-1,-1,1), 5: (1,-1,1), 6: (-1,1,1), 7: (1,1,1)
    Note: The source [S2, unofficial] mentions a 45-degree rotation around X-axis.
    This function should operate on the *transformed* vertices provided in input.
    """
    # Define edge connectivity for a standard cube
    edge_indices = [
        (0,1), (1,3), (3,2), (2,0), # Bottom face
        (4,5), (5,7), (7,6), (6,4), # Top face
        (0,4), (1,5), (2,6), (3,7)  # Vertical edges
    ]
    
    edges = []
    for i, j in edge_indices:
        start = cube_vertices[i]
        end = cube_vertices[j]
        direction = normalize(end - start)
        edges.append((start, direction))
    return edges
```

### Parameters
*   `numberOfSlices` (int, default 10): Number of parallel planes to generate [S2, unofficial].
*   `viewDir` (vec3): Normalized direction vector defining the plane normals and slice ordering [S2, unofficial].
*   `EPSILON` (float): Small constant added/subtracted to min/max distances to prevent missed intersections [S2, unofficial].
*   `planeInc`: The distance between consecutive slice planes, calculated as `PLANE_INC` multiplied by `sliceScale` [S1, unofficial].

### Edge Cases and Pitfalls
*   **Parallel Edges:** When an edge is parallel to the view direction, the denominator $d_i = \text{dot}(\text{edgeDir}[i], V)$ is zero. Division by zero is avoided by checking `1.0 + denom != 1.0`. Parallel edges are skipped by setting $\lambda = -1.0$ and $\lambda_{inc} = 0.0$ [S2, unofficial].
*   **Degenerate Slices:** If the merged vertex count is 2 or fewer, the slice is skipped. This indicates a co-planar slice or no intersection with the bounding box [S1, unofficial].
*   **Precision:** Use `EPSILON` to expand the min/max bounds to ensure that slices at the extreme ends of the volume are not missed due to floating-point inaccuracies [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot E)$, where $N$ is the number of slices and $E$ is the number of edges (constant 12). The intersection calculation is linear in the number of slices.
*   **Memory Complexity:** $O(N \cdot V)$, where $V$ is the number of vertices per slice (up to 6). The geometry grows linearly with the number of slices.
*   **Performance:** Offloads CPU calculation by generating geometry that can be rendered via standard GPU pipelines. The number of triangles is small per slice (up to 8 triangles for 6 vertices), making it efficient for real-time rendering if the number of slices is moderate [S2, unofficial].
*   **Strengths:** Compatible with standard rasterization; allows for easy integration of lighting and shading models per slice; explicit geometry allows for occlusion culling if needed.
*   **Limitations:** Requires explicit geometry management; may produce artifacts if slices are too sparse; back-to-front rendering order must be maintained for correct transparency [S1, unofficial].
*   **When to use:** When explicit geometry is required for volume rendering, or when integrating volume rendering into a pipeline that already handles mesh-based objects.
*   **When not to use:** For very high-resolution volumes where the number of slices would be too large, or when hardware-accelerated ray casting is preferred.

## Background
View-Aligned Slicing, also known as Slice-Base Volume Rendering, is described in *GPU Gems* [S2, unofficial]. It is implemented in open-source projects such as *MedicalViz* [S1, unofficial] and *Slice-Base-Volume-Rendering* [S2, unofficial]. The technique approximates a volume by intersecting a bounding cube with a series of parallel planes aligned with the view direction, generating a polygon mesh per slice. This approach allows the use of standard graphics APIs for rendering volumetric data as a collection of triangles.

## Sources
[S1] DoctorTone/MedicalViz, https://github.com/DoctorTone/MedicalViz, fetched 2026-10-27 (unofficial)
[S2] Conradyen/Slice-Base-Volume-Rendering, https://github.com/Conradyen/Slice-Base-Volume-Rendering, fetched 2026-10-27 (unofficial)
