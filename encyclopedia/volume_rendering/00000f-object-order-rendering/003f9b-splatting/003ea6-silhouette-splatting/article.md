# Silhouette Splatting


## Overview
Silhouette Splatting is a volume rendering technique that projects irregular grid cells by triangulating their convex hulls and rendering them as geometric primitives [S1][S2, unofficial]. Unlike standard splatting which treats data as point samples, this method operates on the cells between sample points [S2, unofficial]. It was developed for the rapid exploration of curvilinear grids but was ultimately abandoned due to visual artifacts and lack of performance gain over simpler methods [S1][S2, unofficial].

## How it works
**Inputs:** A set of irregular grid cells, where each cell is defined by its vertices in 3D space, and associated scalar values (e.g., density or temperature) at those vertices.
**Outputs:** A 2D image.

**Data Structures:**
*   **Cell:** A collection of $N$ vertices $\mathbf{v}_i$ in 3D space.
*   **Convex Hull:** A subset of the cell's vertices that form the outer boundary polygon when viewed from the camera.
*   **Centroid:** The geometric center of the cell, calculated as the average of its vertices.

**Steps:**
1.  **Identify Convex Hull:** Determine which vertices of the current cell lie on the convex hull relative to the current view direction [S1][S2, unofficial].
2.  **Form Polygon:** Connect the convex hull vertices to form a closed polygon [S2, unofficial].
3.  **Compute Centroid:** Calculate the centroid $\mathbf{c}$ of the cell [S2, unofficial].
    $$ \mathbf{c} = \frac{1}{N} \sum_{i=1}^{N} \mathbf{v}_i $$
4.  **Estimate Properties:** Determine the depth $z_c$ and the estimated scalar value $V_c$ (e.g., average of vertex values) for the centroid [S1][S2, unofficial].
5.  **Triangulate:** Create a set of triangles by connecting the centroid $\mathbf{c}$ to each edge of the convex hull polygon [S1][S2, unofficial].
6.  **Project and Render:** Transform the vertices of these triangles into screen space and rasterize them. The depth $z_c$ and value $V_c$ are used for compositing, typically assuming the value is constant across the projected triangle [S1][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def silhouette_splatting(cell, camera):
    # 1. Find vertices on the convex hull
    hull_vertices = get_convex_hull(cell.vertices, camera.view_direction)
    
    # 2. Form polygon from hull vertices
    polygon = Polygon(hull_vertices)
    
    # 3. Find centroid
    centroid = sum(cell.vertices) / len(cell.vertices)
    
    # 4. Estimate depth and value at centroid
    centroid_depth = project_depth(centroid, camera)
    centroid_value = average(cell.vertex_values)
    
    # 5. Triangulate by connecting centroid to hull edges
    triangles = []
    for i in range(len(hull_vertices)):
        v1 = hull_vertices[i]
        v2 = hull_vertices[(i + 1) % len(hull_vertices)]
        # Create triangle from centroid and two adjacent hull vertices
        triangles.append(Triangle(centroid, v1, v2))
        
    # 6. Send triangles to screen with centroid properties
    for tri in triangles:
        tri.depth = centroid_depth
        tri.value = centroid_value
        rasterize(tri, camera)
```

**Edge Cases and Pitfalls:**
*   **Blotchiness:** Images produced are very blotchy from oblique angles on irregular grids [S1][S2, unofficial]. This is because the triangulation from a single centroid does not account for the varying depth and value across the cell surface, leading to discontinuities.
*   **Performance:** The method was not significantly faster than simpler projection methods (like Cell Face Projection with Depth) [S2, unofficial].
*   **Overlap:** Unlike standard splatting, where overlapping splats can reduce blotchiness, this method's rigid triangulation does not easily allow for overlapping primitives without significant additional calculation (e.g., moving the convex hull away from the centroid) [S1][S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by convex hull computation and triangulation per cell. Not significantly faster than cell face projection [S2, unofficial].
*   **Memory Complexity:** Low, as it processes cells individually without complex data structures.
*   **Visual Quality:** Poor on irregular grids, especially at oblique angles, resulting in blotchy images [S1][S2, unofficial].
*   **Strengths:** Simple to implement; conceptually similar to standard splatting but applied to cells.
*   **Limitations:** Visual artifacts; no substantial performance benefit over alternatives; abandoned in favor of other methods [S2, unofficial].
*   **When to use:** Not recommended. The method was abandoned due to poor results [S2, unofficial].

## Background
Silhouette Splatting was proposed in the context of rapid exploration of curvilinear grids using direct volume rendering [S1][S2, unofficial]. It was mentioned primarily to document that the approach was tried and found unsuccessful, saving others from repeating the effort [S2, unofficial]. It is distinct from standard splatting (e.g., [Splatting](23-splatting.md)) because it operates on cells rather than sample points [S2, unofficial].

## Sources
[S1] Rapid Exploration of Curvilinear Grids Using Direct Volume Rendering, https://ntrs.nasa.gov/api/citations/19930015755/downloads/19930015755.pdf, fetched 2026-10-24
[S2] NASA Technical Reports Server (NTRS) 19930015755: Rapid exploration of curvilinear grids using direct volume rendering, https://archive.org/details/NASA_NTRS_Archive_19930015755, fetched 2026-10-26 (unofficial)
