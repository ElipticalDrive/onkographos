# Balanced Cell Projection

Also called: CPU-offloaded Cell Projection, Balanced VICP

## Overview
Balanced Cell Projection is a volume rendering method that optimizes the View Independent Cell Projection (VICP) algorithm by shifting ray-face intersection calculations from the GPU vertex processor to the CPU [S1]. It solves the problem of excessive data transfer between CPU and GPU by pre-computing intersection distances and scalar values on the CPU, passing only minimal results to the GPU for rasterization and fragment processing [S1]. Unlike standard VICP, which relies on vertex shader interpolation of intersection data, Balanced Cell Projection minimizes the payload sent to the GPU, leveraging the CPU's ability to perform bulk geometric calculations without the per-vertex overhead of shader execution [S1].

## How it works
Balanced Cell Projection builds on [Cell projection](26-cell-projection.md) and [View Independent Cell Projection](26-cell-projection.md) (VICP). It retains the core VICP mechanism of determining ray-cell intersections by rasterizing front faces and computing back-face intersections, but changes where the intersection math occurs.

**Inputs and Outputs:**
*   **Input:** Volume data defined on tetrahedral cells; viewing ray direction; camera parameters.
*   **Output:** Composited color and opacity for each pixel.

**Data Structures:**
*   **CPU State:** For each tetrahedron face, store the plane equation coefficients $(A, B, C, D)$ and the scalar gradient $\nabla s = (\frac{\partial s}{\partial x}, \frac{\partial s}{\partial y}, \frac{\partial s}{\partial z})$ [S1].
*   **GPU Payload:** For each vertex of the tetrahedron, store two floating-point values: the scalar value $s$ at the ray-face intersection and the distance $d$ from the vertex to that intersection along the viewing ray [S1].

**Algorithm Steps:**

1.  **Identify Front and Back Faces:** Determine which faces of the tetrahedron are front-facing relative to the camera. Rasterize these front faces to determine the screen-space coverage and the viewing ray intersections with these faces [S1].
2.  **CPU Ray-Face Intersection:** For each vertex of the tetrahedron, calculate the intersection of the viewing ray with the faces sharing that vertex [S1].
    *   A vertex is shared by three of the four faces of the tetrahedron.
    *   The intersection with the face containing the vertex is trivial (the vertex itself) and is skipped [S1].
    *   For the other faces sharing the vertex, compute the intersection of the viewing ray with the face plane.
    *   The plane equation is $Ax + By + Cz + D = 0$. The viewing ray is $P(t) = V + t \cdot R$, where $V$ is the vertex position and $R$ is the ray direction.
    *   Solve for $t$:
        $$ t = -\frac{A V_x + B V_y + C V_z + D}{A R_x + B R_y + C R_z} $$
    *   If $t > 0$, the intersection is valid. Calculate the scalar value $s$ at the intersection point using linear interpolation based on the scalar gradient $\nabla s$ and the distance $t$ [S1].
3.  **Prepare GPU Payload:** For each vertex, store the computed scalar $s$ and distance $t$ (distance-to-vertex) into the GPU payload [S1]. This payload consists of only two floating-point values per vertex, significantly less than the full plane equation and gradient data [S1].
4.  **GPU Rasterization and Fragment Processing:**
    *   Pass the tetrahedron vertices with the pre-computed scalar and distance values to the GPU [S1].
    *   Rasterize the front faces. The GPU interpolates the scalar and distance values across the fragments [S1].
    *   For each fragment, determine the back-face intersection. This is done by intersecting the viewing ray with the plane of each face that is *not* the front face [S1].
    *   Select the rear face as the intersection that is nearest and behind the front face [S1].
    *   Compute the segment length and average differential opacity for the ray segment within the cell.
5.  **Compositing:** Composite the fragment in back-to-front order using the exponential opacity model [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def balanced_cell_projection(tetrahedron, camera_ray, faces):
    """
    tetrahedron: List of 4 vertices [v0, v1, v2, v3]
    camera_ray: Ray object with origin V and direction R
    faces: List of 4 faces, each face is a list of 3 vertex indices
    """
    
    # Step 1: Identify front faces and rasterize them
    # This determines the screen-space polygons covered by the tetrahedron
    front_faces = identify_front_faces(tetrahedron, camera_ray)
    
    # Step 2: CPU Ray-Face Intersection
    # For each vertex, compute intersection with faces sharing it (excluding the face containing the vertex)
    vertex_data = []
    for i, vertex in enumerate(tetrahedron):
        scalar_at_intersection = 0.0
        distance_to_intersection = 0.0
        
        # Find faces sharing this vertex
        shared_faces = [f for f in faces if i in f]
        
        # For each shared face, compute ray-face intersection
        # Skip the face that contains the vertex itself (trivial intersection)
        # Actually, the note says: "at each vertex we need to do only one ray-face intersection"
        # A vertex is shared by 3 faces. One face contains the vertex? No, all 3 contain the vertex.
        # The note says: "For example in Figure 1, vertex v1... at each vertex we need to do only one ray-face intersection."
        # This implies we intersect the ray with the *other* faces sharing the vertex?
        # Let's re-read carefully: "A vertex is shared by three of the four faces... For example... vertex v1... trivial because the intersection is the vertex itself"
        # This suggests that for a given vertex, we intersect the ray with the faces that do NOT contain the vertex?
        # No, the ray originates from the camera. The intersection is with the face plane.
        # The note says: "At each vertex, ray-face intersections with the three faces sharing that vertex are trivial because the intersection is the vertex itself"
        # This is confusing. If the ray passes through the vertex, the intersection is the vertex.
        # But the ray is a viewing ray, it doesn't necessarily pass through the vertex.
        # Let's look at the context: "VICP finds the intersections of the viewing ray with the front faces simply by rasterizing them."
        # "It then determines the back face intersection on a per fragment basis."
        # "To find the back face intersection, VICP intersects the viewing ray with the plane of each face that is not the front face."
        # "So instead, VICP performs these intersections at the vertices in the vertex processor..."
        # "We move operations back into the CPU... Specifically, we perform ray-face intersections on the CPU."
        # "The ray-face intersection determines the distance from a vertex to a face along the viewing ray..."
        # "At each vertex we need to do only one ray-face intersection. A vertex is shared by three of the four faces... trivial because the intersection is the vertex itself"
        # This implies that for a vertex, we intersect the ray with the face that does NOT contain the vertex?
        # No, a tetrahedron has 4 faces. Each vertex is part of 3 faces. There is 1 face that does NOT contain the vertex.
        # So, for each vertex, we intersect the ray with the single face that does NOT contain that vertex.
        
        non_containing_face = None
        for f in faces:
            if i not in f:
                non_containing_face = f
                break
        
        if non_containing_face:
            # Compute intersection of ray with the plane of non_containing_face
            plane = get_plane_equation(non_containing_face, tetrahedron)
            t = intersect_ray_plane(camera_ray, plane)
            
            if t > 0:
                # Calculate scalar value at intersection
                # Scalar gradient is needed
                gradient = get_scalar_gradient(non_containing_face, tetrahedron)
                intersection_point = camera_ray.origin + t * camera_ray.direction
                scalar_at_intersection = interpolate_scalar(intersection_point, gradient, tetrahedron)
                distance_to_intersection = t
        
        vertex_data.append((scalar_at_intersection, distance_to_intersection))
    
    # Step 3: Pass data to GPU
    # Only scalar and distance-to-vertex values are passed
    gpu_payload = vertex_data
    
    # Step 4: GPU Fragment Processing
    # Rasterize front faces with the payload
    # For each fragment, determine back face intersection
    fragments = rasterize_front_faces(front_faces, gpu_payload)
    
    results = []
    for frag in fragments:
        # Determine back face intersection
        # Intersect viewing ray with planes of faces that are not the front face
        back_face_intersection = None
        min_dist = infinity
        
        for face in faces:
            if face not in front_faces:
                plane = get_plane_equation(face, tetrahedron)
                t = intersect_ray_plane(camera_ray, plane)
                if t > 0 and t < min_dist:
                    min_dist = t
                    back_face_intersection = t
        
        # Select rear face as nearest and behind front face
        # Front face distance is known from rasterization (interpolated)
        front_dist = frag.interpolated_distance
        
        if back_face_intersection > front_dist:
            segment_length = back_face_intersection - front_dist
            # Compute opacity
            alpha = compute_opacity(segment_length, frag.scalar)
            results.append((frag.color, alpha))
    
    # Step 5: Composite
    return composite_back_to_front(results)

def intersect_ray_plane(ray, plane):
    """
    plane: (A, B, C, D) for Ax + By + Cz + D = 0
    ray: origin V, direction R
    """
    A, B, C, D = plane
    V = ray.origin
    R = ray.direction
    
    denominator = A * R.x + B * R.y + C * R.z
    if abs(denominator) < epsilon:
        return infinity # Parallel
    
    numerator = -(A * V.x + B * V.y + C * V.z + D)
    return numerator / denominator

def get_plane_equation(face_indices, vertices):
    """
    Compute plane equation from 3 vertices
    """
    v0 = vertices[face_indices[0]]
    v1 = vertices[face_indices[1]]
    v2 = vertices[face_indices[2]]
    
    # Normal vector
    n = cross(v1 - v0, v2 - v0)
    A, B, C = n
    
    # D = -n . v0
    D = -dot(n, v0)
    
    return (A, B, C, D)

def interpolate_scalar(point, gradient, vertices):
    """
    Linear interpolation using gradient
    """
    # s(p) = s(v0) + gradient . (p - v0)
    # Assuming gradient is constant over the cell
    v0 = vertices[0]
    s0 = vertices[0].scalar
    return s0 + dot(gradient, point - v0)
```

**Parameters:**
*   **Plane Equation Coefficients:** $(A, B, C, D)$ [S1].
*   **Scalar Gradient:** $(\frac{\partial s}{\partial x}, \frac{\partial s}{\partial y}, \frac{\partial s}{\partial z})$ [S1].
*   **GPU Payload Size:** 2 floating-point values per vertex (scalar and distance) [S1].

**Edge Cases and Pitfalls:**
*   **Trivial Intersections:** At each vertex, the ray-face intersection with the face containing the vertex is trivial (the vertex itself) and should be skipped to avoid unnecessary computation and data transfer [S1].
*   **Precision:** The method relies on accurate plane equations and gradient calculations on the CPU. Precision errors in these calculations can affect the accuracy of the scalar interpolation and distance computation [S1].
*   **Degenerate Cells:** If a tetrahedron is degenerate (e.g., zero volume), the plane equation may be undefined or the intersection calculation may fail. Standard cell projection techniques for handling degenerate cells should be applied [S1].

## Operating characteristics
*   **Time Complexity:** The CPU performs ray-face intersections for all vertices of all cells. This is $O(N_v)$ where $N_v$ is the number of vertices. The GPU performs rasterization and fragment processing, which is $O(N_p)$ where $N_p$ is the number of pixels covered by the cells.
*   **Memory Complexity:** The CPU stores plane equations and gradients for all faces. The GPU payload is minimal (2 floats per vertex). This reduces memory bandwidth usage compared to VICP [S1].
*   **Measured Performance:** The method reduces data transfer between CPU and GPU, which was a major bottleneck in VICP [S1]. Performance gains depend on the ratio of CPU computation time to data transfer time [S1].
*   **Strengths:** Minimizes data transfer; leverages CPU for bulk geometric calculations; maintains high accuracy [S1].
*   **Limitations:** Requires CPU computation for each frame; may not be suitable for highly dynamic scenes where CPU becomes the bottleneck [S1].
*   **When to Use:** When data transfer bandwidth is a limiting factor and the CPU has sufficient computational power [S1].
*   **When Not to Use:** When the scene changes rapidly and CPU computation time exceeds the benefits of reduced data transfer [S1].

## Background
Balanced Cell Projection was introduced by Kenneth Moreland in "A Fast High Accuracy Volume Renderer for Unstructured Grids" (2004) [S1]. It builds on the View Independent Cell Projection (VICP) algorithm by Weiler, Kraus, and Ertl (2002) [S1]. The motivation was to address the data transfer bottleneck in VICP, where the vertex processor was used to compute ray-face intersections, requiring significant data to be passed to the GPU [S1]. By moving these calculations to the CPU, Moreland reduced the payload to the GPU, balancing the workload between CPU and GPU [S1].

## Sources
[S1] A Fast High Accuracy Volume Renderer for Unstructured ..., https://www.kennethmoreland.com/partial-pre-integration/volvis04.pdf, fetched 2026-10-28
