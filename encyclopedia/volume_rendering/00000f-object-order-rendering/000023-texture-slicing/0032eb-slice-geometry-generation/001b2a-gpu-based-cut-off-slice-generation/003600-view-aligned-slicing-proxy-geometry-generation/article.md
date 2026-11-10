# View-Aligned Slicing (Proxy Geometry Generation)

Also called: Bounding Box Plane Intersection, Cube-Plane Intersection Slicing, Geometric Slice Casting, Planar Slice Volume Rendering, Slice-Base Volume Rendering, Volume Slicing

## Overview
View-Aligned Slicing is a volume rendering technique that approximates a volumetric dataset by intersecting its bounding box with a series of parallel planes aligned with the view direction, generating a polygon mesh for each slice [S2, unofficial]. It solves the problem of CPU-side geometric intersection calculations by offloading the computation of slice vertices to the GPU or pre-computing them for efficient rasterization [S1, unofficial]. Unlike texture-based slicing which samples a 3D texture, this method generates explicit geometry (proxy meshes) that can be rendered using standard triangle pipelines, often facilitating advanced shading models like Layered Polygonal Area Light (LTC) integration [S1, unofficial]. It builds on [GPU-based Cut-off Slice Generation](948-gpu-based-cut-off-slice-generation.md) by explicitly defining the plane parameters and intersection logic.

## How it works
**Inputs:**
*   `cameraPos`: Camera position vector.
*   `target`: Camera target (look-at point) vector.
*   `cube_vertices`: Array of 8 vertices defining the volume's bounding box (typically a unit cube or AABB).
*   `numberOfSlice`: Integer count of slices to generate (default 10) [S2, unofficial].
*   `sliceScale`: Scaling factor for slice spacing [S1, unofficial].

**Outputs:**
*   A list of meshes (or vertex/index buffers), one per slice, ordered back-to-front relative to the view direction.

**Data Structures:**
*   `cube_vertices[8]`: `glm::vec3` array. In [S2, unofficial], these are unit cube vertices transformed by a 45-degree rotation around the X-axis.
*   `lambda[12]`, `lambda_inc[12]`: Float arrays of size 12, storing the intersection parameter and its increment per slice for each of the 12 cube edges.
*   `intersection[6]`: `glm::vec3` temporary array storing up to 6 intersection points per slice before fan assembly.
*   `vTextureSlices`: Container for the final generated slice geometries.

**Parameters:**
*   `viewDir`: Normalized direction vector from target to camera (`cameraPos - target`) [S1, unofficial].
*   `EPSILON`: Small float constant added/subtracted to min/max distances to prevent missed intersections due to precision errors [S2, unofficial].
*   `planeInc`: Distance between consecutive slice planes, calculated as `PLANE_INC * sliceScale` [S1, unofficial].

**Algorithm Steps:**

1.  **Compute View Direction:**
    Calculate `viewDir = normalize(cameraPos - target)` [S1, unofficial].

2.  **Determine Slice Extents:**
    Compute the dot product of `viewDir` with all 8 `cube_vertices` to find the minimum (`min_z`) and maximum (`max_z`) distances along the view axis [S2, unofficial].
    Expand these bounds: `min_z -= EPSILON`, `max_z += EPSILON` [S2, unofficial].
    Find the bounding box vertex closest to the camera to determine the starting slice position [S1, unofficial].
    Calculate the initial plane offset by projecting the closest vertex onto the viewing direction, subtracting the target, and taking the length [S1, unofficial].

3.  **Calculate Slice Count and Spacing:**
    Calculate `planeInc` based on `sliceScale` [S1, unofficial].
    Calculate the number of slices:
    $$ numSlices = \text{round}\left(\frac{planeOffset}{planeInc}\right) \times 2 $$
    where `planeOffset` is the initial distance from the target to the first slice plane [S1, unofficial].
    Alternatively, slice spacing can be derived as `deltaZ = (max_z - min_z) / numberOfSlice` [S2, unofficial].

4.  **Initialize Plane and Edge Intersections:**
    Invert the plane offset and initialize the intersect plane with `viewDir` and the offset [S1, unofficial].
    For each of the 12 cube edges (defined by start vertex `edgeStart[i]` and direction `edgeDir[i]`):
    *   Compute `denom = dot(edgeDir[i], viewDir)` [S2, unofficial].
    *   If `1.0 + denom != 1.0` (i.e., `denom` is not effectively zero):
        $$ \lambda_{inc}[i] = \frac{plane\_dist_{inc}}{denom} $$
        $$ \lambda[i] = \frac{plane\_dist - dot(edgeStart[i], viewDir)}{denom} $$
    *   Else (edge parallel to view):
        $$ \lambda[i] = -1.0, \quad \lambda_{inc}[i] = 0.0 $$
    [S2, unofficial].

5.  **Generate Slices (Back-to-Front):**
    Iterate `i` from `numSlices - 1` down to `0` [S2, unofficial].
    For each edge `e`:
    *   Compute `dL[e] = lambda[e] + i * lambda_inc[e]` [S2, unofficial].
    *   If `dL[e] >= 0.0` and `dL[e] < 1.0`:
        $$ intersection[k] = edgeStart[e] + dL[e] \times edgeDir[e] $$
        Store in `intersection` array, incrementing index `k` [S2, unofficial].

6.  **Triangulate and Assemble:**
    If the number of collected intersection points is 2 or fewer, skip this slice (indicates co-planar or no intersection) [S1, unofficial].
    Use a fixed triangular fan index pattern to form triangles from the `intersection` points [S2, unofficial]:
    *   For 3 vertices: `(0, 1, 2)`
    *   For 4 vertices: `(0, 1, 2), (0, 2, 3)`
    *   For 5 vertices: `(0, 1, 2), (0, 2, 3), (0, 3, 4)`
    *   For 6 vertices: `(0, 1, 2), (0, 2, 3), (0, 3, 4), (0, 4, 5)`
    [S1, S2, unofficial]. Note: [S1, unofficial] specifies slightly different fan indices `(0,2,1)` etc., but the topology is equivalent (fan from vertex 0). The standard fan order is used here for clarity.
    Append vertices and indices to `vTextureSlices` [S2, unofficial].

7.  **Finalize Geometry:**
    Merge duplicate vertices in the geometry to clean up the point cloud [S1, unofficial].
    Compute bounding sphere and face normals for the triangulated geometry [S1, unofficial].
    Create a mesh from the geometry and shader.
    Set render order to `numSlices - slice` to ensure correct back-to-front drawing [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def view_aligned_slicing(camera_pos, target, cube_vertices, num_slices, slice_scale, epsilon):
    """
    Generates proxy geometry for volume slicing aligned with the view direction.
    
    Args:
        camera_pos: vec3, camera position
        target: vec3, camera look-at target
        cube_vertices: list of 8 vec3, bounding box vertices
        num_slices: int, number of slices to generate
        slice_scale: float, scaling factor for plane spacing
        epsilon: float, small constant for boundary expansion
        
    Returns:
        list of Mesh objects, ordered back-to-front
    """
    # 1. Compute View Direction
    view_dir = normalize(camera_pos - target)
    
    # 2. Determine Extents
    # Project all cube vertices onto view_dir
    projections = [dot(v, view_dir) for v in cube_vertices]
    min_z = min(projections) - epsilon
    max_z = max(projections) + epsilon
    
    # Find closest vertex to camera (largest projection if view_dir is camera->target, 
    # but typically view_dir is target->camera in graphics, so closest is min projection?
    # Source [S1, unofficial] says "closest to camera... initial plane offset". 
    # Source [S2, unofficial] uses min/max z for range.
    # We assume standard view direction: camera to target or target to camera.
    # Let's assume view_dir points from camera to target (standard GL).
    # Then closest vertex to camera has MIN projection.
    closest_proj = min(projections)
    
    # Plane spacing
    plane_inc = slice_scale # Assuming PLANE_INC is 1.0 or absorbed in scale [S1, unofficial]
    
    # Calculate number of slices if not provided, or use provided
    # Formula from [S1, unofficial]: numSlices = round(planeOffset / planeInc) * 2
    # Here we use the input num_slices or calculate from range
    if num_slices == 0:
        plane_offset = abs(closest_proj) # Distance from target to first slice?
        num_slices = int(round(plane_offset / plane_inc) * 2)
        
    delta_z = (max_z - min_z) / num_slices
    
    # 3. Precompute Edge Intersections
    # Define 12 edges of the cube. 
    # Note: cube_vertices must be ordered consistently for edges.
    # Assuming standard cube vertex ordering or explicit edge list.
    edges = get_cube_edges(cube_vertices) # Returns list of (start, dir) tuples
    
    lambda_vals = [0.0] * 12
    lambda_inc_vals = [0.0] * 12
    
    for i, (edge_start, edge_dir) in enumerate(edges):
        denom = dot(edge_dir, view_dir)
        if 1.0 + denom != 1.0: # Check for non-zero
            # plane_dist_inc is effectively delta_z in the projection space
            lambda_inc_vals[i] = delta_z / denom
            # Initial lambda for slice 0
            # plane_dist is min_z
            lambda_vals[i] = (min_z - dot(edge_start, view_dir)) / denom
        else:
            lambda_vals[i] = -1.0
            lambda_inc_vals[i] = 0.0
            
    slices = []
    
    # 4. Generate Slices Back-to-Front
    # Iterate from last slice to first
    for i in range(num_slices - 1, -1, -1):
        intersections = []
        
        for e in range(12):
            dL = lambda_vals[e] + i * lambda_inc_vals[e]
            
            if 0.0 <= dL < 1.0:
                edge_start, edge_dir = edges[e]
                pt = edge_start + dL * edge_dir
                intersections.append(pt)
                
        # 5. Triangulate
        if len(intersections) <= 2:
            continue # Skip degenerate slices
            
        # Fan triangulation from vertex 0
        indices = []
        for k in range(1, len(intersections) - 1):
            indices.append(0)
            indices.append(k)
            indices.append(k + 1)
            
        # Create Mesh
        mesh = create_mesh(intersections, indices)
        mesh.render_order = i # Back-to-front
        slices.append(mesh)
        
    return slices

def get_cube_vertices():
    """Returns 8 unit cube vertices rotated 45 deg around X axis [S2, unofficial]"""
    # Standard unit cube
    base = [
        [-1, -1, -1], [1, -1, -1], [1, 1, -1], [-1, 1, -1],
        [-1, -1, 1], [1, -1, 1], [1, 1, 1], [-1, 1, 1]
    ]
    # Rotation matrix for 45 deg around X
    cos_a = cos(pi/4)
    sin_a = sin(pi/4)
    rotated = []
    for v in base:
        x, y, z = v
        # Rotate around X
        ny = y * cos_a - z * sin_a
        nz = y * sin_a + z * cos_a
        rotated.append([x, ny, nz])
    return rotated

def get_cube_edges(vertices):
    """Returns 12 edges as (start_vertex, direction_vector)"""
    # Standard cube edge connectivity
    edge_indices = [
        (0,1), (1,2), (2,3), (3,0), # Bottom
        (4,5), (5,6), (6,7), (7,4), # Top
        (0,4), (1,5), (2,6), (3,7)  # Vertical
    ]
    edges = []
    for i, j in edge_indices:
        start = vertices[i]
        end = vertices[j]
        dir_vec = normalize(end - start)
        edges.append((start, dir_vec))
    return edges
```

## Subtopics

### [Volume Domain Transformation for Rectangular Slicing](11012-volume-domain-transformation-for-rectangular-slicing.md)
A preprocessing step that transforms the AABB volume domain into a transformed domain to ensure all cutting sections are rectangles, simplifying LTC-based integration and avoiding complex polygon clipping. It builds on [View-Aligned Slicing (Proxy Geometry Generation)](7620-view-aligned-slicing-proxy-geometry-generation.md).

## Operating characteristics
*   **Time Complexity:** $O(N \cdot E)$ per frame, where $N$ is the number of slices and $E$ is the number of edges (constant 12). The intersection calculation is linear in the number of slices.
*   **Memory Complexity:** $O(N \cdot V)$, where $V$ is the number of vertices per slice (up to 6). Geometry is generated per slice.
*   **Performance:** Offloads intersection logic from CPU to GPU or pre-computation, enabling real-time rendering of complex volumes [S2, unofficial]. Performance depends heavily on the number of slices and the cost of the subsequent shading (e.g., LTC integration) [S1, unofficial].
*   **Strengths:** Explicit geometry allows for advanced shading techniques not easily achievable with texture slicing (e.g., accurate normal-based lighting, LTC) [S1, unofficial]. Robust handling of view-aligned planes [S2, unofficial].
*   **Limitations:** Generates N-gons (up to hexagons) which must be triangulated, potentially causing artifacts if not handled correctly [S1, unofficial]. Requires careful handling of degenerate cases (parallel edges, no intersection) [S1, S2, unofficial].
*   **When to use:** When advanced per-slice shading is required, or when the volume is viewed from a fixed or slowly changing direction allowing pre-computation.
*   **When not to use:** For highly dynamic viewpoints where pre-computation is infeasible and GPU vertex shader intersection is too costly, or when the volume is extremely large and slice count must be minimal.

## Background
The method is described in the context of slice-based volume rendering, often associated with implementations in graphics libraries and educational projects [S2, unofficial]. It is referenced in the context of Medical Visualization tools [S1, unofficial]. The technique is related to the "Slice-Base Volume Rendering" approach described in *GPU Gems* [S2, unofficial].

## Sources
[S1] DoctorTone/MedicalViz, https://github.com/DoctorTone/MedicalViz, fetched 2026-10-27 (unofficial)
[S2] Conradyen/Slice-Base-Volume-Rendering, https://github.com/Conradyen/Slice-Base-Volume-Rendering, fetched 2026-10-27 (unofficial)
