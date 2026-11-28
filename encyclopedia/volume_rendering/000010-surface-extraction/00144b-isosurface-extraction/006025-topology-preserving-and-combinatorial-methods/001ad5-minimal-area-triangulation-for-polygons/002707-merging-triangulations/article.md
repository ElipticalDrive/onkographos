# Merging Triangulations

Also called: Stitching Triangulations

## Overview
Merging Triangulations is the final assembly step in hierarchical surface reconstruction methods that use adaptive clustering and piecewise polynomial approximation [S1, unofficial]. It combines individual optimal triangulations generated for separate cluster regions into a single triangulation by filling the gaps along cluster boundaries defined by a BSP tree hierarchy [S1, unofficial]. Unlike methods that operate directly on raw data points, this approach exploits the connectivity of the clustering hierarchy to stitch pre-computed triangulations [S1, unofficial]. It is distinguished from sibling methods by its reliance on a reverse-order subdivision process and its specific handling of boundary polygons to maintain geometric consistency across cluster interfaces [S1, unofficial].

## How it works
The method operates on a set of individual optimal triangulations, each associated with a cluster region in a BSP tree hierarchy [S1, unofficial]. The goal is to merge these disjoint triangulations into a single global triangulation [S1, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   A BSP tree defining the cluster hierarchy [S1, unofficial].
    *   A set of individual optimal triangulations, one for each leaf cluster region [S1, unofficial].
    *   For each triangulation, a closed boundary polygon enclosing all triangles, edges, and vertices within the cluster region [S1, unofficial].
    *   An error bound $\epsilon$ for the approximation [S1, unofficial].
*   **Outputs:**
    *   A single unified triangulation covering the entire domain [S1, unofficial].
    *   Triangles that satisfy the error bound (or are modified to do so) [S1, unofficial].

### Data Structures
*   **Boundary Polygon:** A list of vertices forming a closed loop [S1, unofficial]. The polygon encloses all simplices (triangles, edges, vertices) of the optimal triangulation within the cluster [S1, unofficial]. It does not enclose areas other than optimal triangles [S1, unofficial]. Vertices may appear multiple times in the list if they are part of the boundary loop multiple times [S1, unofficial].
*   **Polygon Strip:** A subsequence of the boundary polygon vertices extracted along the cluster bisector [S1, unofficial].

### Steps

1.  **Construct Boundary Polygons:**
    For each optimal triangulation, construct a closed boundary polygon [S1, unofficial].
    *   For parabolic or planar graph surfaces, this is straightforward [S1, unofficial].
    *   For elliptic or hyperbolic cases, triangulations may be disconnected or contain line segments not part of any triangle [S1, unofficial]. Special handling is required to ensure the polygon encloses all simplices [S1, unofficial].
    *   **Algorithm:** Use a scan-line algorithm parallel to the longest edge of the triangulation [S1, unofficial]. Identify the left-most and right-most vertices inside the cluster region for every scan line [S1, unofficial]. Add additional vertices inside the cluster region to the boundary polygon to ensure the enclosed area contains *only* optimal triangles, minimizing the enclosed area [S1, unofficial].

2.  **Shrink Cluster Regions:**
    Shrink each cluster region by a small percentage towards its centroid to avoid boundary overlaps during stitching [S1, unofficial].

3.  **Merge in Reverse Subdivision Order:**
    Traverse the BSP tree in reverse order of subdivision (from leaves to root) [S1, unofficial]. At each internal node, exactly two child triangulations are merged along the bisecting line segment $p_1p_2$ that separates their cluster regions [S1, unofficial].

4.  **Extract Polygon Strips:**
    For the two triangulations to be merged, separated by segment $p_1p_2$:
    *   Identify the vertex in the first boundary polygon closest to $p_1$ [S1, unofficial].
    *   Identify the vertex in the second boundary polygon closest to $p_2$ [S1, unofficial].
    *   Extract two polygon strips from the respective boundaries, starting from the identified vertices and ending at the vertices closest to $p_2$ and $p_1$ respectively [S1, unofficial]. These strips lie along the bisector $p_1p_2$ [S1, unofficial].

5.  **Marching and Triangle Construction:**
    Construct triangles by marching along the two polygon strips, connecting point pairs using vertices from opposite sides of $p_1p_2$ [S1, unofficial].
    *   Let the current vertices on the two strips be $u$ and $v$ [S1, unofficial].
    *   There are two possible choices for the next triangle: $(u, v, next_u)$ or $(u, v, next_v)$, where $next_u$ and $next_v$ are the subsequent vertices on their respective strips [S1, unofficial].
    *   **Selection Criterion:** Choose the edge with minimal length among the two possible choices [S1, unofficial]. Specifically, compare the length of the edge connecting to the next vertex on the opposite strip [S1, unofficial]. Prefer the shorter edge [S1, unofficial].

6.  **Handle Negative Area (Overlap):**
    Ensure no triangles with negative area (overlap) are produced [S1, unofficial].
    *   If both choices for the next triangle would result in negative area, construct a triangle from three consecutive vertices in one of the polygon strips, thereby eliminating the middle vertex [S1, unofficial].
    *   If stitching still produces negative areas for both choices after this step, insert a triangle from three vertices of the *other* polygon strip [S1, unofficial].

7.  **Error Verification and Correction:**
    Triangles resulting from stitching are not guaranteed to satisfy the prescribed error bound $\epsilon$ [S1, unofficial].
    *   Estimate the approximation error for all scattered data points located inside the regions defined by the newly stitched triangles [S1, unofficial].
    *   If a triangle violates the error bound:
        *   Flip an edge between two adjacent triangles, or
        *   Insert the sample point with the greatest approximation error as a new vertex into the triangulation [S1, unofficial].
    *   Re-check errors for all modified triangles and repeat until all triangles satisfy the bound [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Triangle:
    def __init__(self, v0, v1, v2):
        self.v = (v0, v1, v2)

def signed_area(p1, p2, p3):
    """Returns 2x the signed area of the triangle."""
    return (p2[0] - p1[0]) * (p3[1] - p1[1]) - (p2[1] - p1[1]) * (p3[0] - p1[0])

def distance(p1, p2):
    return ((p1[0] - p2[0])**2 + (p1[1] - p2[1])**2)**0.5

def extract_polygon_strip(boundary_polygon, start_point, end_point):
    """
    Extracts a polygon strip from the boundary polygon.
    Starts at the vertex closest to start_point and ends at the vertex closest to end_point.
    """
    start_idx = min(range(len(boundary_polygon)), key=lambda v: distance(boundary_polygon[v], start_point))
    end_idx = min(range(len(boundary_polygon)), key=lambda v: distance(boundary_polygon[v], end_point))
    
    strip = []
    n = len(boundary_polygon)
    curr = start_idx
    while True:
        strip.append(boundary_polygon[curr])
        if curr == end_idx:
            break
        curr = (curr + 1) % n
    return strip

def estimate_max_error(triangle, data_points):
    """Estimates the maximum approximation error for data points inside the triangle. [G]"""
    # Standard approach: evaluate residual at data points inside triangle and return max norm.
    max_err = 0.0
    for pt in data_points:
        if point_in_triangle(pt, triangle.v):
            err = abs(approximation_residual(pt, triangle))
            if err > max_err:
                max_err = err
    return max_err

def point_in_triangle(pt, verts):
    """Barycentric coordinate test."""
    v0, v1, v2 = verts
    d1 = signed_area(pt, v1, v2)
    d2 = signed_area(v0, pt, v2)
    d3 = signed_area(v0, v1, pt)
    has_neg = (d1 < 0) or (d2 < 0) or (d3 < 0)
    has_pos = (d1 > 0) or (d2 > 0) or (d3 > 0)
    return not (has_neg or has_pos)

def approximation_residual(pt, triangle):
    """Returns the approximation error at point pt. [G]"""
    # Placeholder for the specific quadratic polynomial evaluation defined by the cluster.
    return 0.0

def try_edge_flip(triangle, triangulation):
    """Attempts to flip an edge of the triangle to reduce error. [G]"""
    # Standard Delaunay-style or error-driven edge flip logic.
    # Returns True if flip was successful and improved error.
    return False

def find_worst_approximation_point(triangle, data_points):
    """Finds the data point inside the triangle with the greatest approximation error. [G]"""
    worst_pt = None
    max_err = -1.0
    for pt in data_points:
        if point_in_triangle(pt, triangle.v):
            err = abs(approximation_residual(pt, triangle))
            if err > max_err:
                max_err = err
                worst_pt = pt
    return worst_pt

def insert_vertex_into_triangle(triangle, vertex, triangulation):
    """Inserts a new vertex into the triangle, splitting it into three sub-triangles. [G]"""
    # Standard vertex insertion: replace triangle with three new triangles sharing the new vertex.
    pass

def merge_triangulations(bsp_tree, triangulations):
    """
    Merges individual optimal triangulations into a single mesh.
    
    Args:
        bsp_tree: The BSP tree defining cluster hierarchy.
        triangulations: Dictionary mapping cluster_id to list of Triangle objects.
        
    Returns:
        Unified list of Triangle objects.
    """
    nodes = bsp_tree.get_nodes_in_reverse_subdivision_order()
    
    for node in nodes:
        if node.is_leaf():
            continue
            
        left_child = node.left_child
        right_child = node.right_child
        
        p1, p2 = node.get_bisector_endpoints()
        
        tri_left = triangulations[left_child.id]
        tri_right = triangulations[right_child.id]
        
        # Shrink cluster regions slightly towards centroid to avoid overlap
        tri_left.shrink_boundary(0.01)
        tri_right.shrink_boundary(0.01)
        
        strip_left = extract_polygon_strip(tri_left.boundary_polygon, p1, p2)
        strip_right = extract_polygon_strip(tri_right.boundary_polygon, p2, p1)
        
        new_triangles = []
        i = 0
        j = 0
        
        while i < len(strip_left) - 1 and j < len(strip_right) - 1:
            u = strip_left[i]
            v = strip_right[j]
            u_next = strip_left[i+1]
            v_next = strip_right[j+1]
            
            len_cross_1 = distance(u, v_next)
            len_cross_2 = distance(v, u_next)
            
            area_1 = signed_area(u, v, v_next)
            area_2 = signed_area(u, v, u_next)
            
            if area_1 < 0 and area_2 < 0:
                # Both choices produce negative area
                # Construct triangle from three consecutive vertices in one strip
                # Eliminate the middle vertex. Prefer one strip arbitrarily or based on heuristic.
                next_next_u = strip_left[i+2] if i+2 < len(strip_left) else u
                new_triangles.append(Triangle(u, u_next, next_next_u))
                i += 2
                continue
            
            if area_1 < 0:
                # Choice 1 invalid, must use Choice 2
                new_triangles.append(Triangle(u, v, u_next))
                i += 1
                j += 1
                continue
                
            if area_2 < 0:
                # Choice 2 invalid, must use Choice 1
                new_triangles.append(Triangle(u, v, v_next))
                i += 1
                j += 1
                continue
                
            # Both valid, choose minimal length edge
            if len_cross_1 < len_cross_2:
                new_triangles.append(Triangle(u, v, v_next))
                i += 1
                j += 1
            else:
                new_triangles.append(Triangle(u, v, u_next))
                i += 1
                j += 1
                
        combined_tri = tri_left + tri_right + new_triangles
        
        # Error verification and correction
        combined_tri = correct_error_bounds(combined_tri, node.cluster_data_points, node.error_bound)
        
        triangulations[node.id] = combined_tri
        
    return triangulations[bsp_tree.root.id]

def correct_error_bounds(triangulation, data_points, error_bound):
    """
    Checks error bounds for stitched triangles and modifies if necessary.
    """
    modified = True
    while modified:
        modified = False
        for triangle in triangulation:
            max_error = estimate_max_error(triangle, data_points)
            
            if max_error > error_bound:
                if try_edge_flip(triangle, triangulation):
                    modified = True
                    continue
                
                worst_point = find_worst_approximation_point(triangle, data_points)
                if worst_point is not None:
                    insert_vertex_into_triangle(triangle, worst_point, triangulation)
                    modified = True
                
    return triangulation
```

### Parameters
*   **Error Bound Split:** In numerical examples, a greater error bound is used for the quadratic approximation step than for the final triangulation of polynomials [S1, unofficial]. This makes the total error bound relatively loose for the triangles obtained from the stitching process, making violations of the error estimate extremely rare [S1, unofficial].
*   **Shrink Percentage:** A small percentage (e.g., 1%) used to shrink cluster regions towards their centroid to prevent boundary overlaps [S1, unofficial].

### Edge Cases and Pitfalls
*   **Negative Area (Overlap):** The algorithm must strictly avoid producing triangles with negative area [S1, unofficial]. If both standard choices for the next triangle result in negative area, a triangle must be constructed from three consecutive vertices in one polygon strip, eliminating the middle vertex [S1, unofficial]. If this still fails, a triangle from three vertices of the other polygon strip is inserted [S1, unofficial].
*   **Disconnected Triangulations:** In elliptic and hyperbolic cases, the optimal triangulation inside a cluster region may be disconnected or contain line segments not part of any triangle [S1, unofficial]. The boundary construction must handle these cases to ensure the boundary polygon correctly encloses all simplices [S1, unofficial].
*   **Vertex Multiplicity:** A vertex can be part of the boundary polygon multiple times [S1, unofficial]. The algorithm must handle this correctly when extracting strips and marching [S1, unofficial].

## Subtopics

### Optimal Approximation with Triangles
A hierarchical triangulation approach that uses piecewise quadratic polynomial approximation as an intermediate representation [S1, unofficial]. It generates optimally shaped triangles for each cluster region such that all triangles imply the same approximation error [S1, unofficial]. Unlike methods operating directly on data points, this method is based on adaptive clustering and quadratic representation [S1, unofficial]. The cost depends on clustering complexity and error bound, not the number of original data points [S1, unofficial]. It serves as the precursor to Merging Triangulations, providing the individual optimal triangulations that are then stitched together [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The cost of generating the triangulation depends on the complexity of the clustering and the error bound, not on the number of points in the original data set [S1, unofficial].
*   **Memory Complexity:** Requires storing the boundary polygons for each cluster and the BSP tree hierarchy [S1, unofficial].
*   **Strengths:**
    *   Clusters remain invariant under translation, rotation, or uniform scaling [S1, unofficial].
    *   Produces optimally shaped triangles with minimal area for a given error bound [S1, unofficial].
    *   Efficient for densely sampled, smooth data sets [S1, unofficial].
*   **Limitations:**
    *   Requires a hierarchical clustering scheme (e.g., BSP tree) [S1, unofficial].
    *   Error bounds for stitched triangles are not guaranteed a priori and may require correction steps [S1, unofficial].
    *   Clustering schemes are not invariant under affine transformations like shear, though an affine-invariant norm can be exploited via PCA transformation [S1, unofficial].
*   **When to Use:** Suitable for scientific visualization of scattered data where approximation error is critical and the data is densely sampled and smooth [S1, unofficial].

## Background
The method was introduced in the context of multiresolution modeling for scientific visualization [S1, unofficial]. It builds upon earlier work on optimal approximation with triangles and adaptive clustering schemes [S1, unofficial]. The approach exploits the affine invariance of the error norm and the connectivity of a BSP tree to efficiently stitch triangulations [S1, unofficial]. Numerical examples were provided for analytical functions and terrain data sets [S1, unofficial]. The method is part of a broader family of hierarchical surface reconstruction techniques that aim to balance geometric fidelity with computational efficiency [S1, unofficial].

## Sources
[S1] Multiresolution modeling for scientific visualization, https://escholarship.org/uc/item/5317385v, fetched 2026-10-26 (unofficial)
