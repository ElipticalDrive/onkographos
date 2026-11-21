# Slice Vertex Sorting via Pseudo-Angle

Also called: Pseudo-Angle Metric Sorting, Pseudo-Angle Sorting

## Overview
This method sorts the intersection vertices of a volume slice around their centroid in counter-clockwise order using a custom pseudo-angle metric [S1, unofficial]. It solves the problem of generating a valid triangle fan or polygon from unordered intersection points, which is a necessary step after [GPU-based Cut-off Slice Generation](948-gpu-based-cut-off-slice-generation.md) produces the raw vertex data. It differs from [Incremental Intersection and Texture Coordinates](997-incremental-intersection-and-texture-coordinates.md) and [View-Aligned Slicing (Proxy Geometry Generation)](7620-view-aligned-slicing-proxy-geometry-generation.md) by relying on a post-hoc geometric sort of the entire vertex set rather than incremental traversal or proxy geometry alignment [S1, unofficial].

## How it works
**Inputs:** A list of 3D vertices $V = \{v_0, v_1, \dots, v_{n-1}\}$ representing the intersection polygon of a slice with the volume bounding box.
**Outputs:** A sorted list of vertices $V_{sorted}$ ordered counter-clockwise around the centroid, ready for triangle fan generation.

**Data Structures:**
- `vertices`: A dynamic array or vector of 3D points (floats).
- `angles`: A parallel array or struct array storing the computed pseudo-angle for each vertex.

**Steps:**
1. **Compute Centroid:** Calculate the centroid $C$ by averaging all vertex coordinates:
   $$ C = \frac{1}{n} \sum_{i=0}^{n-1} v_i $$
2. **Compute Pseudo-Angle:** For each vertex $v_i$, calculate the delta vector $\Delta = v_i - C$. Compute the pseudo-angle $A(\Delta)$ using the formula below.
3. **Sort:** Sort the vertices based on their computed pseudo-angles in ascending order.
4. **Generate Geometry:**
   - If $n > 3$, generate triangles by iterating through the sorted vertices, creating triangles formed by $v_{sorted}[i]$, $C$, and $v_{sorted}[(i+1)\%n]$.
   - If $n \le 3$, push the vertices in sorted order directly as a polygon (or degenerate triangles).

**Formula:**
The pseudo-angle metric $A(p)$ for a delta vector $p = (\Delta x, \Delta y)$ is defined as [S1, unofficial]:

$$
A(p) = \begin{cases} 
-1 & \text{if } \Delta x = 0 \text{ and } \Delta y = 0 \\
2.0 - \frac{\Delta y}{|\Delta x| + |\Delta y|} & \text{if } \Delta x < 0 \\
4.0 + \frac{\Delta y}{|\Delta x| + |\Delta y|} & \text{if } \Delta x \ge 0 
\end{cases}
$$

Where $\Delta x$ and $\Delta y$ are the x and y components of the vector $p = v_i - C$. Note that this metric maps the 2D plane (projected onto the slice's local coordinate system, typically the view plane) into a range that allows simple numerical sorting to achieve angular ordering. The value $-1$ is used for the centroid itself if it coincides with a vertex, ensuring it sorts first or last depending on implementation, though typically the centroid is not part of the vertex list being sorted.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_pseudo_angle(delta_x, delta_y):
    """
    Computes the pseudo-angle for a 2D delta vector.
    Returns a float value that can be sorted to achieve CCW ordering.
    """
    if delta_x == 0.0 and delta_y == 0.0:
        return -1.0
    
    denominator = abs(delta_x) + abs(delta_y)
    result = delta_y / denominator
    
    if delta_x < 0.0:
        result = 2.0 - result
    else:
        result = 4.0 + result
        
    return result

def sort_vertices_ccw(vertices):
    """
    Sorts a list of 3D vertices counter-clockwise around their centroid
    using the pseudo-angle metric.
    
    Args:
        vertices: List of glm::vec3 or similar 3D point objects.
        
    Returns:
        Sorted list of vertices.
    """
    if not vertices:
        return []
        
    n = len(vertices)
    
    # Step 1: Compute Centroid
    centroid_x = sum(v.x for v in vertices) / n
    centroid_y = sum(v.y for v in vertices) / n
    # Note: z-component of centroid is not used in the 2D pseudo-angle calculation
    # as the sorting is performed on the slice plane coordinates.
    
    # Step 2: Compute Pseudo-Angles
    # Store tuples of (angle, original_index) to keep track of original vertices
    angle_list = []
    for i, v in enumerate(vertices):
        delta_x = v.x - centroid_x
        delta_y = v.y - centroid_y
        angle = compute_pseudo_angle(delta_x, delta_y)
        angle_list.append((angle, i))
        
    # Step 3: Sort by angle
    angle_list.sort(key=lambda x: x[0])
    
    # Step 4: Retrieve sorted vertices
    sorted_vertices = [vertices[idx] for _, idx in angle_list]
    
    return sorted_vertices

def generate_slice_geometry(sorted_vertices, centroid):
    """
    Generates triangle indices or vertices for the slice.
    """
    n = len(sorted_vertices)
    triangles = []
    
    if n > 3:
        # Generate triangle fan from centroid
        for i in range(n):
            v1 = sorted_vertices[i]
            v2 = sorted_vertices[(i + 1) % n]
            # Triangle: v1, centroid, v2
            triangles.append((v1, centroid, v2))
    else:
        # For n <= 3, just return vertices in order
        # Or generate degenerate triangles if required by the renderer
        return sorted_vertices
        
    return triangles
```

**Parameters:**
- No tunable parameters. The metric is deterministic.

**Edge Cases and Pitfalls:**
- **Centroid Coincidence:** If a vertex lies exactly on the centroid, $\Delta x = 0$ and $\Delta y = 0$, returning $-1$. This vertex will sort to the beginning of the list. In a triangle fan, this may cause degenerate triangles if not handled, but typically the centroid is a separate point, not part of the vertex list $V$.
- **Precision:** Floating-point precision issues may cause slight ordering errors for vertices very close to the axes or centroid. However, for volume rendering slices, this is generally acceptable.
- **2D Projection:** The formula uses only $\Delta x$ and $\Delta y$. This implies the vertices are already projected onto the 2D slice plane (typically the view plane). If the slice is not aligned with the view, the vertices must be transformed into the slice's local 2D coordinate system before applying this metric. The source code uses `glm::vec3` but the formula is 2D, suggesting the z-component is ignored or the input is already 2D [S1, unofficial].

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(n \log n)$ due to sorting, where $n$ is the number of intersection vertices. Computing the centroid and angles is $O(n)$.
- **Memory Complexity:** $O(n)$ to store the angles and sorted indices.
- **Strengths:** Simple to implement; does not require complex geometric predicates or cross-product calculations for ordering.
- **Limitations:** Relies on a 2D projection; if the slice plane is not aligned with the coordinate system used for the pseudo-angle, the sorting will be incorrect. It assumes the intersection polygon is convex or star-shaped with respect to the centroid.
- **When to use:** When the number of vertices per slice is small and a simple, robust sorting method is preferred over incremental traversal.

## Background
This method is implemented in the open-source project "Slice-Base-Volume-Rendering" by Conradyen [S1, unofficial]. It is used as a utility function within the rendering pipeline to prepare slice geometry for rasterization.

## Sources
[S1] Conradyen/Slice-Base-Volume-Rendering, https://github.com/Conradyen/Slice-Base-Volume-Rendering, fetched 2026-10-27 (unofficial)
