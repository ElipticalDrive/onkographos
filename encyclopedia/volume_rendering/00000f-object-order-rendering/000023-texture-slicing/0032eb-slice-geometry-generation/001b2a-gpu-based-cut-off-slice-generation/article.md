# GPU-based Cut-off Slice Generation

Also called: Plane-Cube Clipping, Rezk-Salama method, Slice-Cube Intersection, Vertex Shader Slice Generation, View-Aligned Slicing (Proxy Geometry), ViewSliceCube, Viewport-Aligned Slices Intersection Algorithm

## Overview
GPU-based Cut-off Slice Generation is an object-order volume rendering technique that computes the intersection polygon between a slicing plane and the volume's bounding box directly within a vertex shader [S1, unofficial][S2, book][S4, unofficial]. It solves the problem of expensive CPU-side geometric calculations by offloading the determination of proxy geometry (polygonal meshes) to the GPU, enabling real-time interactive slicing [S1, unofficial][S2, book]. It differs from sibling methods by using a fixed triangular fan assembly based on precomputed edge intersections and a specific vertex ordering strategy (Rezk-Salama) rather than incremental texture coordinate updates or pseudo-angle sorting [S1, unofficial][S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Volumetric data (bound as 3D textures), camera parameters (position, view direction $\vec{n}_p$), modelview matrix, number of slices ($N$), and bounding box definition (typically a unit cube or axis-aligned box).
*   **Outputs:** A stream of triangle primitives representing the cross-sections of the volume, textured with the 3D volume data.

**Data Structures:**
*   **Bounding Box Vertices:** 8 vertices defining the volume extent.
*   **Edge Definitions:** 12 edges, each defined by a start vertex $\vec{V}_i$ and direction $\vec{e}_{i \rightarrow j} = \vec{V}_j - \vec{V}_i$.
*   **Edge Flags:** For each edge, a 6-bit integer where each bit corresponds to one of the 6 faces of the bounding box. A bit is set if the edge belongs to that face [S3, unofficial].
*   **Output Array:** A contiguous buffer (e.g., `renderSliceArray`) storing vertices. For each slice, it stores up to 12 vertices (4 triangles) [S4, unofficial].

**Algorithm Steps:**

1.  **Coordinate Transformation:** Transform the 8 bounding box vertices from object space to view space using the modelview matrix [S2, book][S5].
2.  **Determine Slicing Extent:** Compute the minimum ($min\_z$) and maximum ($max\_z$) dot products of the transformed vertices with the view direction $\vec{n}_p$ (typically $(0,0,-1)$ in view space) [S4, unofficial].
    *   Apply epsilon expansion to avoid missed intersections: $min\_z \leftarrow min\_z - \epsilon$, $max\_z \leftarrow max\_z + \epsilon$ [S4, unofficial].
    *   Calculate slice spacing: $plane\_dist\_inc = (max\_z - min\_z) / N$ [S4, unofficial].
    *   Initialize current plane distance: $plane\_dist = min\_z$ [S4, unofficial].
3.  **Intersection Calculation (Per Slice):** For each slice plane at distance $plane\_dist$:
    *   For each of the 12 edges, compute the intersection parameter $\lambda$ [S1, unofficial]:
        $$ \lambda = \begin{cases} \frac{plane\_dist - \vec{n}_p \cdot \vec{V}_i}{\vec{n}_p \cdot \vec{e}_{i \rightarrow j}}, & \text{if } \vec{n}_p \cdot \vec{e}_{i \rightarrow j} \neq 0 \\ -1.0, & \text{otherwise} \end{cases} $$
        *   If $\lambda \in [0, 1]$, the intersection is valid and lies on the edge [S1, unofficial].
        *   If the denominator is zero (edge parallel to plane), set $\lambda = -1.0$ [S1, unofficial].
    *   **Vertex Ordering and Polygon Construction:** The algorithm must output exactly 6 vertices per slice to form a closed polygon, even if the actual cross-section has fewer vertices. This is achieved by grouping edges into 6 specific traversal groups (Red, Yellow, Green, Cyan, Blue, Purple) based on their connectivity to the nearest and farthest vertices of the bounding box [S1, unofficial].
        *   **Red Group:** Edges connecting the nearest vertex to the farthest vertex via one intermediate vertex (e.g., $E_{0\rightarrow1}, E_{1\rightarrow4}, E_{4\rightarrow7}$). These always have exactly one intersection.
        *   **Blue Group:** Edges connecting the nearest vertex to the farthest vertex via the other path (e.g., $E_{0\rightarrow3}, E_{3\rightarrow6}, E_{6\rightarrow7}$). These always have exactly one intersection.
        *   **Green Group:** Edges connecting the nearest vertex to the farthest vertex via the third path (e.g., $E_{0\rightarrow2}, E_{2\rightarrow5}, E_{5\rightarrow7}$). These always have exactly one intersection.
        *   **Yellow, Cyan, Purple Groups:** Remaining edges. These may have 0 or 1 intersection.
        *   **Traversal Order:** Red $\rightarrow$ Yellow $\rightarrow$ Green $\rightarrow$ Cyan $\rightarrow$ Blue $\rightarrow$ Purple [S1, unofficial].
        *   **Handling Missing Intersections:** If an edge in the Yellow, Cyan, or Purple groups has no valid intersection ($\lambda \notin [0,1]$), the algorithm duplicates the intersection point from the preceding Red, Green, or Blue edge respectively [S1, unofficial]. This ensures the output vertex list always contains 6 points forming a valid closed polygon (triangle, quad, pentagon, or hexagon) [S1, unofficial].
4.  **Texture Coordinate Computation:** For each valid intersection point, compute texture coordinates $(u,v,w)$ to sample the 3D volume [S1, unofficial]. Using the flexible coordinate translation by Shen et al. [S1, unofficial]:
    $$ PI \cdot \vec{k} = \begin{cases} \frac{V_i \vec{k} - min(B_{\vec{k}})}{max(B_{\vec{k}}) - min(B_{\vec{k}})} & \text{if } \vec{k} \cdot \vec{e}_{i \rightarrow j} = 0 \\ \lambda & \text{if } \vec{k} \cdot \vec{e}_{i \rightarrow j} > 0 \\ 1 - \lambda & \text{if } \vec{k} \cdot \vec{e}_{i \rightarrow j} < 0 \end{cases} $$
    where $\vec{k}$ is the x, y, or z unit vector in texture space, and $B_{\vec{k}}$ is the bounding box length in direction $\vec{k}$ [S1, unofficial].
5.  **Tessellation:** Triangulate the 6-vertex polygon. A common method is to fan from the first vertex (index 0) to the others: indices $\{0,1,2\}, \{0,2,3\}, \{0,3,4\}, \{0,4,5\}$ [S4, unofficial].
6.  **Rendering:** Pass the vertices to the fragment shader. The fragment shader samples the 3D texture at the interpolated texture coordinates and applies a transfer function (color/opacity lookup) [S1, unofficial]. Slices are composited using alpha blending in back-to-front or front-to-back order [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
EPSILON = 0.0001
NUM_EDGES = 12
NUM_VERTICES_BOX = 8

# Edge definitions: (start_vertex_index, end_vertex_index)
# Assuming standard cube vertex ordering:
# 0:(0,0,0), 1:(1,0,0), 2:(0,1,0), 3:(0,0,1), 4:(1,1,0), 5:(1,0,1), 6:(0,1,1), 7:(1,1,1)
# Note: Specific indices depend on the bounding box definition. 
# The Rezk-Salama method relies on specific topological groups.
EDGES = [
    (0, 1), (1, 4), (4, 7), # Red Group 1
    (0, 3), (3, 6), (6, 7), # Blue Group 1
    (0, 2), (2, 5), (5, 7), # Green Group 1
    (1, 5), (2, 6), (3, 4)  # Yellow, Cyan, Purple Groups
]

# Mapping of edge index to its group for ordering:
# 0,1,2 -> Red; 3,4,5 -> Blue; 6,7,8 -> Green; 9 -> Yellow; 10 -> Cyan; 11 -> Purple
# The output order is Red(0,1,2), Yellow(9), Green(6,7,8), Cyan(10), Blue(3,4,5), Purple(11)
OUTPUT_ORDER = [0, 1, 2, 9, 6, 7, 8, 10, 3, 4, 5, 11]

def compute_slice_geometry(box_vertices_view, view_dir, plane_dist, num_slices):
    """
    Computes the 6 intersection points for a single slice plane.
    box_vertices_view: list of 8 vec3, transformed to view space.
    view_dir: vec3, typically (0,0,-1).
    plane_dist: float, distance of the plane from origin along view_dir.
    """
    intersections = []
    
    # Precompute edge directions and dot products
    edge_data = []
    for i in range(NUM_EDGES):
        v_start = box_vertices_view[EDGES[i][0]]
        v_end = box_vertices_view[EDGES[i][1]]
        edge_dir = v_end - v_start
        denom = dot(view_dir, edge_dir)
        
        if abs(denom) < 1e-9: # Parallel to plane
            lambda_val = -1.0
            lambda_inc = 0.0 # Not used for single slice, but for incremental
        else:
            # lambda = (plane_dist - n . Vi) / (n . e)
            lambda_val = (plane_dist - dot(view_dir, v_start)) / denom
            
        edge_data.append({
            'start': v_start,
            'dir': edge_dir,
            'lambda': lambda_val
        })

    # Collect valid intersections in the specific Rezk-Salama order
    # We need exactly 6 points. If an edge has no intersection, duplicate the previous valid one.
    
    valid_points = []
    
    # Helper to add point if valid
    def add_point_if_valid(edge_idx):
        ed = edge_data[edge_idx]
        if 0.0 <= ed['lambda'] <= 1.0:
            pt = ed['start'] + ed['lambda'] * ed['dir']
            valid_points.append(pt)
            return True
        return False

    # Traverse in order: Red (0,1,2), Yellow (9), Green (6,7,8), Cyan (10), Blue (3,4,5), Purple (11)
    # Note: The source [S1, unofficial] describes groups. 
    # Red: E0->1, E1->4, E4->7 (Indices 0, 1, 2 in EDGES list if defined as above)
    # Blue: E0->3, E3->6, E6->7 (Indices 3, 4, 5)
    # Green: E0->2, E2->5, E5->7 (Indices 6, 7, 8)
    # Remaining: E1->5 (9), E2->6 (10), E3->4 (11) -> Yellow, Cyan, Purple
    
    # We iterate through the 12 edges in the specific output order.
    # If an edge yields a valid intersection, we add it.
    # If it does NOT yield a valid intersection, we duplicate the LAST added valid point.
    
    final_points = []
    
    # Define the traversal sequence of edge indices
    traversal_sequence = [0, 1, 2, 9, 6, 7, 8, 10, 3, 4, 5, 11]
    
    last_valid_point = None
    
    for edge_idx in traversal_sequence:
        ed = edge_data[edge_idx]
        if 0.0 <= ed['lambda'] <= 1.0:
            pt = ed['start'] + ed['lambda'] * ed['dir']
            final_points.append(pt)
            last_valid_point = pt
        else:
            # No intersection on this edge. Duplicate the previous valid intersection.
            # This handles the "missing" vertices for triangles/quads.
            if last_valid_point is not None:
                final_points.append(last_valid_point)
            else:
                # Should not happen if the plane intersects the box, 
                # but if it misses entirely, we might need to handle degenerate cases.
                # For a valid slice, at least one point should exist.
                pass
                
    # Ensure we have 6 points. If the plane is tangent or near a corner, 
    # the logic above might produce fewer if not careful, but Rezk-Salama 
    # guarantees 6 by duplication.
    while len(final_points) < 6:
        if last_valid_point:
            final_points.append(last_valid_point)
        else:
            break # Degenerate case
            
    return final_points[:6]
```

## Subtopics

### [Incremental Intersection and Texture Coordinates](997-incremental-intersection-and-texture-coordinates.md)
Computes intersections incrementally for adjacent slices, leveraging uniform spacing to update positions via constant deltas, avoiding recomputation.

### [View-Aligned Slicing (Proxy Geometry Generation)](7620-view-aligned-slicing-proxy-geometry-generation.md)
A broader category of GPU-based proxy geometry generation, often using fixed triangular fans and precomputed edge intersections, differing from incremental methods.

### [Slice Vertex Sorting via Pseudo-Angle](7642-slice-vertex-sorting-via-pseudo-angle.md)
Sorts intersection vertices around a centroid using a pseudo-angle metric to ensure correct triangulation order, a prerequisite for rendering.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot E)$ per frame, where $N$ is the number of slices and $E$ is the number of edges (constant 12). The calculation is performed in parallel on the GPU vertex shader, making it highly efficient for interactive rates [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for storing the slice geometry buffers. The output buffer size is fixed per slice (e.g., 12 vertices for 4 triangles) [S4, unofficial].
*   **Performance:** Allows real-time interaction with response times < 100ms [S1, unofficial]. Performance scales with the number of slices and resolution of the 3D texture, not the volume data size directly [S1, unofficial].
*   **Strengths:** Offloads CPU, enables real-time rotation and slicing, simple implementation in vertex shaders [S1, unofficial].
*   **Limitations:** Limited to convex bounding boxes (typically cubes). Cannot handle complex internal structures or non-convex volumes without additional logic. Duplicate vertices must be handled to maintain fixed output size [S1, unofficial].

## Background
The method originates from the work of Rezk-Salama et al. [50], who proposed moving the calculation of cut-off slices from the CPU to the GPU vertex shader [S1, unofficial]. It was further adapted and popularized in systems like the one described by Rui [56] and implemented in various real-time volume rendering frameworks [S1, unofficial]. It is a standard technique in object-order volume rendering for interactive visualization [S2, book].

## Sources
[S1] A system for real-time rendering of compressed time-varying volume data, https://doi.org/10.7939/r3041m, fetched 2026-10-26 (unofficial)
[S2] Chapter 39. Volume Rendering Techniques | NVIDIA Developer, https://developer.nvidia.com/gpugems/gpugems/part-vi-beyond-triangles/chapter-39-volume-rendering-techniques, fetched 2026-10-26 (book)
[S3] Volume rendering techniques for general purpose graphics hardware, https://opus4.kobv.de/opus4-fau/files/9/Dissertation.pdf, fetched 2026-10-26 (unofficial)
[S4] Conradyen/Slice-Base-Volume-Rendering, https://github.com/Conradyen/Slice-Base-Volume-Rendering, fetched 2026-10-27 (unofficial)
[S5] GPU Gems ch39, Volume rendering techniques, https://developer.nvidia.com/sites/all/modules/custom/gpugems/books/GPUGems/gpugems_ch39.html, fetched 2026-10-29
