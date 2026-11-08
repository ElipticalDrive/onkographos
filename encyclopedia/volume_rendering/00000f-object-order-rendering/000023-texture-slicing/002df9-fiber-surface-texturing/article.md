# Fiber Surface Texturing

Also called: FSCP Texturing, Fiber Surface Enhancement via Texture Mapping

## Overview
Fiber Surface Texturing is a method for visualizing the internal structure of fiber surfaces (specifically Fiber Surface Cross-Section Polygons, or FSCP) within tetrahedral meshes by mapping a 1D texture pattern onto the surface geometry [S1]. It solves the problem of displaying continuous fiber paths and their topological changes across a discretized surface without requiring complex per-vertex shading logic [S1]. It is distinguished from standard surface texturing by its specific parameterization strategy, which maps the global fiber path length onto a single texture coordinate, allowing a "dotted line" texture to reveal fiber presence and continuity [S1].

## How it works
**Inputs:**
*   A Fiber Surface Cross-Section Polygon (FSCP) mesh, composed of triangles representing the intersection of fibers with a tetrahedral mesh.
*   A 1D texture map containing a pattern (e.g., alternating black and white segments) representing fiber presence [S1].

**Outputs:**
*   A rasterized image where fibers are visible as lines on the surface, with color/opacity determined by the texture map [S1].

**Data Structures:**
*   **FSCP Mesh:** A set of triangles. Each triangle vertex is associated with a specific segment of the fiber path.
*   **Texture Map:** A 1D texture (or 2D with height 1) where the horizontal axis corresponds to the global fiber parameter $t \in [0, 1]$. Black values indicate a fiber segment; white values indicate no fiber [S1].

**Steps:**
1.  **Segment Identification:** Identify the specific fiber segment $uv$ associated with each triangle in the FSCP. The algorithm processes each segment separately [S1].
2.  **Local Parameter Assignment:** For each triangle belonging to segment $uv$, assign local texture coordinates $u$ and $v$ to the vertices. Specifically, assign $u=0$ to the vertex at the start of the segment and $v=1$ to the vertex at the end of the segment [S1].
3.  **Global Parameter Mapping:** Map the local parameters to a global range $[0, 1]$ representing the total length of the fiber path. This is done by scaling the local $u,v$ values based on the segment's position in the overall fiber sequence [S1].
    *   Let $t_{global}$ be the global parameter for a vertex.
    *   If a vertex is at the start of segment $i$, $t_{global} = \sum_{j=0}^{i-1} L_j / L_{total}$, where $L_j$ is the length of segment $j$.
    *   The local parameter is then interpolated within this global range [S1].
4.  **Texture Sampling:** During rasterization, sample the 1D texture map using the computed global texture coordinate $t_{global}$ for each fragment [S1].
5.  **Color Application:** Apply the sampled color (black for fiber, white for no fiber) to the fragment [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fiber_surface_texturing(fscp_mesh, texture_map):
    """
    fscp_mesh: List of triangles. Each triangle has a reference to a fiber segment.
    texture_map: 1D array of colors, indexed by global parameter t in [0, 1].
    """
    # Precompute global segment boundaries
    segments = fscp_mesh.get_unique_segments()
    total_length = sum(seg.length for seg in segments)
    
    # Calculate global start and end parameters for each segment
    segment_globals = []
    current_t = 0.0
    for seg in segments:
        seg_start_t = current_t
        seg_end_t = current_t + seg.length / total_length
        segment_globals.append((seg.id, seg_start_t, seg_end_t))
        current_t = seg_end_t

    for triangle in fscp_mesh.triangles:
        seg_id = triangle.segment_id
        # Find global bounds for this segment
        seg_start_t, seg_end_t = get_global_bounds(segment_globals, seg_id)
        
        # Assign texture coordinates to vertices
        # Vertex 0 is at start of segment, Vertex 1 is at end
        # Note: The source specifies assigning u=0, v=1 to vertices of the segment.
        # In a triangle, we need to map these to the three vertices.
        # Assuming the triangle lies entirely within the segment uv.
        
        t0 = seg_start_t  # Vertex at u=0
        t1 = seg_end_t    # Vertex at v=1
        t2 = seg_end_t    # Third vertex also at v=1 (or interpolated if triangle spans differently, 
                          # but source implies segment-based assignment) [S1]
        
        # Interpolate t for the third vertex if necessary, 
        # but source emphasizes u=0, v=1 for the segment endpoints.
        # For a triangle within the segment, all vertices map to the range [seg_start_t, seg_end_t].
        
        for vertex in triangle.vertices:
            # Determine local position within the segment
            # If vertex is the 'u' vertex, local_t = 0
            # If vertex is the 'v' vertex, local_t = 1
            # If vertex is intermediate, interpolate.
            # Source simplifies: "assign u, v the parameters 0, 1 respectively"
            # and "map each vertex' parameter to the global range".
            
            local_t = get_local_parameter(vertex, triangle) # 0.0 or 1.0 or interpolated
            global_t = seg_start_t + local_t * (seg_end_t - seg_start_t)
            
            # Sample texture
            color = texture_map.sample(global_t)
            vertex.color = color
```

**Edge Cases and Pitfalls:**
*   **Sharp Bends:** The method reveals inaccuracies in the original fiber surface algorithm at sharp bends in the polygon, where the texture mapping may not align perfectly with the geometric curvature [S1].
*   **Precision:** Mapping to a global $[0, 1]$ range requires careful handling of floating-point precision to avoid texture bleeding or gaps between segments [S1].

## Subtopics

### [KRS Texturing](10157-krs-texturing.md)
Texturing method for KRS using u,v parametrization for simple kernel solids, or procedural/triplanar texturing for complex or reflective kernels.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of triangles in the FSCP mesh, as each triangle requires parameter calculation and texture sampling [S1].
*   **Memory Complexity:** Low, requiring only the FSCP mesh and a small 1D texture map [S1].
*   **Strengths:** Simple to implement using standard texture hardware; effectively visualizes fiber continuity and topological changes [S1].
*   **Limitations:** Accuracy is dependent on the underlying FSCP generation; sharp bends may show artifacts [S1].

## Background
The method was introduced in the context of visualizing fiber surfaces for tetrahedral meshes, specifically to enhance the display of fibers using texture hardware [S1]. It builds on the concept of Fiber Surface Cross-Section Polygons (FSCP) [S1].

## Sources
[S1] Fast and Exact Fiber Surfaces for Tetrahedral Meshes, https://hal.science/hal-01206120v2, fetched 2026-10-26
