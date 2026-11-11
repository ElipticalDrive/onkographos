# Plane-Cube Intersection
Also called: Plane-Cube Intersection Volume, Volume from a plane slicing a cube

## Overview
The Plane-Cube Intersection method computes the geometric intersection between an infinite plane and a cube, resulting in an inscribed irregular polygon with 3 to 6 vertices [S1, unofficial]. It is primarily used to calculate the volume of the portion of the cube sliced by the plane, a computation relevant in computational astrophysics for determining partial cell coverage (e.g., by burning fronts) and in solid modeling [S1, unofficial]. Unlike standard ray-primitive intersection which returns a single point or segment, this method constructs the full cross-sectional polygon to enable volume integration [S1, unofficial].

## How it works
**Inputs:**
*   Cube vertices $\mathbf{V}_i$ (typically a unit cube, but the method generalizes).
*   Plane defined by a unit normal vector $\mathbf{\hat{n}}$ and a scalar distance $dl$ from the origin.

**Outputs:**
*   An ordered list of intersection points forming the polygon vertices.
*   The volume of the polyhedron defined by the plane and the cube.

**Data Structures:**
*   **Plane Definition:** The plane is defined using spherical angles $\theta$ and $\phi$ and a length $dl$. The normal vector $\mathbf{n}$ (before normalization) is:
    $$ \mathbf{n} = [dl \cos(\theta), \ dl \sin(\theta)\sin(\phi), \ dl \cos(\phi) ] $$
    The unit normal is $\mathbf{\hat{n}} = \mathbf{n} / \|\mathbf{n}\|$ [S1, unofficial].
    The plane equation is:
    $$ dl = \mathbf{\hat{n}} \cdot \mathbf{x} $$
    where $\mathbf{x}$ is a point on the plane [S1, unofficial].

*   **Edge Representation:** Each edge $E_{i \rightarrow j}$ connects vertex $\mathbf{V}_i$ to $\mathbf{V}_j$ and is parameterized by $\lambda$:
    $$ E_{i \rightarrow j}(\lambda) = \mathbf{V}_i + \lambda (\mathbf{V}_j - \mathbf{V}_i) $$
    where $0 \le \lambda \le 1$ [S1, unofficial].

**Steps:**
1.  **Define Plane:** Calculate $\mathbf{\hat{n}}$ from $\theta, \phi, dl$ using the spherical coordinate conversion [S1, unofficial].
2.  **Iterate Edges:** For each of the 12 edges of the cube, defined by vertex pairs $(\mathbf{V}_i, \mathbf{V}_j)$:
    a. Calculate the denominator $D = \mathbf{\hat{n}} \cdot (\mathbf{V}_j - \mathbf{V}_i)$.
    b. If $D \approx 0$, the edge is parallel to the plane; skip or handle as degenerate [G].
    c. Calculate the intersection parameter $\lambda$:
       $$ \lambda = \frac{ dl - \mathbf{\hat{n}} \cdot \mathbf{V}_i }{ \mathbf{\hat{n}} \cdot (\mathbf{V}_j - \mathbf{V}_i ) } $$
    d. **Validation:** Check if $0 \le \lambda \le 1$. If true, the plane intersects the edge segment within the cube. Compute the intersection point $\mathbf{P} = \mathbf{V}_i + \lambda (\mathbf{V}_j - \mathbf{V}_i)$ [S1, unofficial].
3.  **Construct Polygon:** Collect all valid intersection points. The method requires constructing an *ordered* list of these face vertices to compute the volume [S1, unofficial]. The number of vertices ranges from 3 to 6 [S1, unofficial].
4.  **Compute Volume:** Use the ordered vertices to calculate the volume of the resulting irregular polyhedron [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def plane_cube_intersection(cube_vertices, theta, phi, dl):
    # 1. Define plane normal
    # n is the unnormalized vector from spherical coords
    n_x = dl * cos(theta)
    n_y = dl * sin(theta) * sin(phi)
    n_z = dl * cos(phi)
    
    # Normalize to get unit normal n_hat
    norm = sqrt(n_x**2 + n_y**2 + n_z**2)
    n_hat = (n_x / norm, n_y / norm, n_z / norm)
    
    intersection_points = []
    
    # 2. Iterate over all 12 edges of the cube
    # Edges are defined by pairs of vertex indices (i, j)
    edges = get_cube_edges(cube_vertices) 
    
    for V_i, V_j in edges:
        # Edge vector
        edge_vec = (V_j.x - V_i.x, V_j.y - V_i.y, V_j.z - V_i.z)
        
        # Denominator: n_hat . (V_j - V_i)
        denom = dot(n_hat, edge_vec)
        
        # Avoid division by zero (parallel edge)
        if abs(denom) < epsilon:
            continue
            
        # Numerator: dl - n_hat . V_i
        # Note: The source defines plane as dl = n_hat . x
        # So dl - n_hat . V_i is the offset of V_i from the plane along normal
        num = dl - dot(n_hat, V_i)
        
        # Calculate lambda
        lam = num / denom
        
        # 3. Validation: Check if intersection is within the edge segment
        if 0.0 <= lam <= 1.0:
            # Compute intersection point
            P_x = V_i.x + lam * edge_vec.x
            P_y = V_i.y + lam * edge_vec.y
            P_z = V_i.z + lam * edge_vec.z
            intersection_points.append((P_x, P_y, P_z))
            
    # 4. Order vertices and compute volume
    # The source states an ordered list is required for volume calculation.
    # The specific ordering algorithm is not detailed in the provided text.
    ordered_vertices = order_polygon_vertices(intersection_points)
    volume = compute_polyhedron_volume(ordered_vertices, cube_vertices)
    
    return ordered_vertices, volume
```

**Parameters:**
*   $\theta$: Azimuthal angle, $0 \le \theta \le 2\pi$ [S1, unofficial].
*   $\phi$: Polar angle, $0 \le \phi \le \pi$ [S1, unofficial].
*   $dl$: Length of the normal ray (distance from origin to plane) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Parallel Edges:** If $\mathbf{\hat{n}} \cdot (\mathbf{V}_j - \mathbf{V}_i) = 0$, the edge is parallel to the plane. The source does not specify handling for coplanar edges, only that intersection requires $0 \le \lambda \le 1$ [S1, unofficial].
*   **Vertex Ordering:** The source emphasizes that constructing an *ordered* list of vertices is critical for volume computation, but does not provide the specific sorting algorithm [S1, unofficial].
*   **Polygon Complexity:** The number of vertices in the intersecting polygon can vary between 3 and 6, showing "wild patterns" with respect to $\theta$ and $\phi$ [S1, unofficial].

## Operating characteristics
*   **Complexity:** The method iterates over a fixed number of cube edges (12), making it $O(1)$ with respect to scene complexity, but dependent on the cube's topology.
*   **Performance:** The source does not provide measured performance metrics or hardware specifics [S1, unofficial].
*   **Strengths:** Provides exact geometric volume for partial cell coverage in astrophysical simulations [S1, unofficial].
*   **Limitations:** Requires explicit vertex ordering for volume calculation, which is not trivial for irregular polygons [S1, unofficial]. The number of vertices changes discontinuously, which may complicate gradient-based optimizations [S1, unofficial].

## Background
The method was presented by F.X. Timmes in the context of computational astrophysics, specifically for calculating the fraction of a cell that is partially ionized or covered by a burning front [S1, unofficial]. It extends the idea from Salama & Kolb (2005) by explicitly constructing an ordered list of face vertices to compute the volume of the irregular polyhedron resulting from the plane slicing the cube [S1, unofficial]. The tool `cube_and_plane.tbz` is provided as a reference implementation [S1, unofficial].

## Sources
[S1] volume of intersection between a plane and a cube - cococubed, https://cococubed.com/code_pages/raybox.shtml, fetched 2026-10-29 (unofficial)
