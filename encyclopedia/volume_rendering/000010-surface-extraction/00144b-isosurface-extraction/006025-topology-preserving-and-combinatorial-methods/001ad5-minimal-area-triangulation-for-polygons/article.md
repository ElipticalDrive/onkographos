# Minimal Area Triangulation for Polygons

Also called: Minimal Area Triangulation, Minimal Area Triangulation of Isopolygons, Polygon Triangulation

## Overview
Minimal Area Triangulation is a method for converting non-convex or complex 3D polygons (isopolygons) resulting from isosurface extraction into a set of triangles, minimizing the total area of the resulting mesh to reduce visual artifacts [S2, S3, unofficial]. It solves the inter-slice triangulation problem by finding the optimal triangle decomposition for isosurfaces defined on octrees [S1, S3, unofficial]. Unlike heuristic methods that prioritize speed or simplicity, this method finds the global optimum for the triangulation of a single polygon [S1, unofficial]. It is distinct from [Advancing Front Framework for Implicit Surfaces](819-advancing-front-framework-for-implicit-surfaces.md) which builds surfaces incrementally, and [Dual Contouring and Surface Nets](6319-dual-contouring-and-surface-nets.md) which use optimization to place vertices, by strictly operating on the boundary polygon of an octree node to produce a minimal-area disk homeomorphic to the input [S1, S3, unofficial].

## How it works
**Inputs:** A simple closed 3D polygon $P$ defined by an ordered list of $n$ vertices $\{p_1, p_2, \dots, p_n\}$ [S3, unofficial]. The polygon lies on the boundary of a convex body (typically an octree node) [S3, unofficial].
**Outputs:** A list of triangles forming the minimal area triangulation of $P$ [S1, unofficial].

**Data Structures:**
1. **Vertices:** Array of 3D points $p_i = (x_i, y_i, z_i)$ [G].
2. **DP Table M:** A 2D array of size $n \times n$ storing the minimal area $M(i, j)$ for sub-polygons [S1, unofficial].
3. **Split Table K:** A 2D array of size $n \times n$ storing the index $k$ that minimized the area for each sub-polygon, used for reconstruction [S1, unofficial].

**Formulas:**
Let $A(i, j, k)$ be the area of the triangle formed by vertices $p_i, p_j, p_k$ [S1, unofficial].
Let $M(i, j)$ be the minimal area of the sub-polygon $P_{i,j}$ defined by vertices from index $i$ to $j$ (inclusive, in cyclic order) [S1, unofficial].

The recurrence relation is [S1, S3, unofficial]:
$$
M(i, j) = 
\begin{cases} 
0 & \text{if } i = j \\
0 & \text{if } i = j + 1 \bmod n \\
\min_{k \in (j, i)} \{ M(i, k) + M(k, j) + A(i, j, k) \} & \text{otherwise}
\end{cases}
$$
Where $k \in (j, i)$ denotes the set of vertices strictly between $j$ and $i$ in the cyclic order of the polygon [S1, S3, unofficial].

**Algorithm Steps:**

1. **Check for Cofacial Vertices:** Determine if all vertices of the polygon lie on the same cube face (are cofacial) [S2, unofficial].
    * If **yes**: Perform a fan triangulation from the centroid to avoid introducing new edges on the cube face, which can cause artifacts [S2, unofficial].
        * Calculate centroid $C = \frac{1}{n} \sum_{i=1}^{n} p_i$ [S2, unofficial].
        * Create triangles $\{C, p_i, p_{i+1}\}$ for $i=1 \dots n-1$ (with $p_{n+1} = p_1$) [S2, unofficial].
        * Return these triangles.
    * If **no**: Proceed to dynamic programming [S2, unofficial].

2. **Initialize DP Tables:** Initialize $M(i, j) = 0$ for all $i, j$ [S1, unofficial]. Initialize $K(i, j)$ to undefined [G].

3. **Compute Minimal Areas (Bottom-Up):** Iterate over the chain length $L$ from 2 to $n-1$ [G]. For each starting index $i$ from 0 to $n-1$: Calculate ending index $j = (i + L) \bmod n$ [G]. Initialize $min\_val = \infty$ and $best\_k = -1$ [G]. Iterate over all vertices $k$ strictly between $j$ and $i$ in the cyclic order [S1, unofficial]. Calculate $current\_val = M(i, k) + M(k, j) + A(i, j, k)$ [S1, unofficial]. If $current\_val < min\_val$: set $min\_val = current\_val$ and $best\_k = k$ [G]. Set $M(i, j) = min\_val$ and $K(i, j) = best\_k$ [S1, unofficial].

4. **Reconstruct Triangles:** Initialize an empty list `triangles` [G]. Call recursive function `Reconstruct(i, j)` starting with $i=0, j=n-1$ (representing the full polygon) [G]. **Reconstruct(i, j):** If $j = (i + 1) \bmod n$: The edge $(i, j)$ is a boundary edge of the original polygon. No triangle is formed by the split at this level [S1, unofficial]. Return. Let $k = K(i, j)$. Add triangle $\{p_i, p_j, p_k\}$ to `triangles` [S1, unofficial]. Recursively call `Reconstruct(i, k)` and `Reconstruct(k, j)` [G].

5. **Return:** The list `triangles` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def minimal_area_triangulation(vertices):
    n = len(vertices)
    if n < 3:
        return []

    # Check for cofacial vertices [S2, unofficial]
    if is_cofacial(vertices):
        return fan_triangulation(vertices)

    # Initialize DP tables [S1, S3, unofficial]
    # M[i][j] stores minimal area for sub-polygon from i to j
    # K[i][j] stores the optimal split vertex k
    M = [[0.0] * n for _ in range(n)]
    K = [[-1] * n for _ in range(n)]

    # Helper to get area of triangle (i, j, k) [G]
    def triangle_area(i, j, k):
        p1 = vertices[i]
        p2 = vertices[j]
        p3 = vertices[k]
        v1 = [p2[x] - p1[x] for x in range(3)]
        v2 = [p3[x] - p1[x] for x in range(3)]
        cross = [
            v1[1] * v2[2] - v1[2] * v2[1],
            v1[2] * v2[0] - v1[0] * v2[2],
            v1[0] * v2[1] - v1[1] * v2[0]
        ]
        return 0.5 * (cross[0]**2 + cross[1]**2 + cross[2]**2)**0.5

    # Fill DP table bottom-up [S1, S3, unofficial]
    # L is the number of steps along the boundary from i to j.
    # Base cases L=0 (i=j) and L=1 (j=i+1) are already 0.
    for L in range(2, n):
        for i in range(n):
            j = (i + L) % n
            
            min_val = float('inf')
            best_k = -1
            
            # Iterate k strictly between j and i in cyclic order [S1, S3, unofficial]
            k = (j + 1) % n
            while k != i:
                val = M[i][k] + M[k][j] + triangle_area(i, j, k)
                if val < min_val:
                    min_val = val
                    best_k = k
                k = (k + 1) % n
            
            M[i][j] = min_val
            K[i][j] = best_k

    # Reconstruct triangles [S1, S3, unofficial]
    triangles = []
    
    def reconstruct(i, j):
        if j == (i + 1) % n:
            return
        
        k = K[i][j]
        triangles.append((i, j, k))
        reconstruct(i, k)
        reconstruct(k, j)

    # Start reconstruction at the full polygon [G]
    reconstruct(0, n - 1)
    
    return triangles

def is_cofacial(vertices):
    # Check if all vertices lie on the same coordinate plane (e.g., x=const, y=const, or z=const) [G]
    if len(vertices) < 3:
        return False
    for axis in range(3):
        val = vertices[0][axis]
        if all(v[axis] == val for v in vertices):
            return True
    return False

def fan_triangulation(vertices):
    n = len(vertices)
    centroid = [sum(v[axis] for v in vertices) / n for axis in range(3)]
    triangles = []
    # Create triangles connecting each edge of the polygon to the centroid [S2, unofficial]
    for i in range(n):
        triangles.append((i, (i + 1) % n, 'centroid'))
    return triangles
```

**Parameters:**
*   None explicitly required for the core algorithm [G].

**Edge Cases and Pitfalls:**
*   **Cofacial Vertices:** If vertices lie on the same cube face, standard triangulation might create edges that lie on the face of the cube, potentially causing artifacts [S2, unofficial].
*   **Self-Intersections:** While the method finds the minimal area *triangulation*, it does not guarantee a minimal area *surface* [S3, unofficial].
*   **Precision:** The dynamic programming approach relies on floating-point arithmetic for area calculations [G].

## Subtopics

### [Fuchs-Kedem-Uselton Contour Method](1780-fuchs-kedem-uselton-contour-method.md)
The Fuchs-Kedem-Uselton Contour Method is an isosurface extraction technique that generates a minimum-area triangulated surface by connecting contours from adjacent volume slices. It solves the inter-slice triangulation problem by modeling the connection between two contours as a minimum-cost cycle in a directed-toroidal graph. Unlike heuristic methods that prioritize speed over optimality, this method finds the optimal solution to generate the minimum-area surface.

### [Merging Triangulations](3936-merging-triangulations.md)
Merging Triangulations is the final assembly step in hierarchical surface reconstruction methods that use adaptive clustering and piecewise polynomial approximation. It combines individual optimal triangulations generated for separate cluster regions into a single triangulation by filling the gaps along cluster boundaries defined by a BSP tree hierarchy. Unlike methods that operate directly on raw data points, this approach exploits the connectivity of the clustering hierarchy to stitch pre-computed triangulations. It is distinguished from sibling methods by its reliance on a reverse-order subdivision process and its specific handling of boundary polygons to maintain geometric consistency across cluster interfaces.

### [Complex Pressure Field Visualization](10459-complex-pressure-field-visualization.md)
Complex Pressure Field Visualization is a technique for visualizing simulated acoustics by rendering "valley surfaces" to represent node regions (areas of sound cancellation) in complex-valued sound pressure fields obtained from Finite Element Method (FEM) simulations. It solves the problem of analyzing wave behavior and sound properties in rooms by highlighting minima that persist over time or allowing selective slicing-plane based visualizations. It sets itself apart from general isosurface methods by specifically targeting the complex-valued nature of acoustic eigenmodes, using mesh voxelization and boolean operations to accumulate and intersect node structures across time steps.

### Afront
Afront uses Catmull-Rom spline for surface approximation on the voxelized grid [S4]. It is controlled by the parameter Rho ($\rho$), which controls triangulation size; reducing $\rho$ by half is roughly equivalent to reducing the maximum triangle size by half [S4]. Another parameter, Eta ($\eta$), exists but its influence was not analyzed in the source [S4].

## Operating characteristics
**Time Complexity:** $O(n^3)$, where $n$ is the number of vertices in the polygon [S1, S3, unofficial].
**Space Complexity:** $O(n^2)$, for the caching array $M$ and $K$ [S1, S3, unofficial].
**Performance:** The method is optimal for the triangulation of a single polygon but can be computationally expensive for polygons with a large number of vertices [G].
**Strengths:** Produces the minimal area triangulation for the given polygon, reducing visual artifacts compared to non-optimal triangulations [S3, unofficial].
**Limitations:** Does not guarantee a minimal area surface (only minimal area triangulation) [S3, unofficial]. Self-intersections are not theoretically prevented, though rare in practice [S3, unofficial]. Cofacial vertices require special handling [S2, unofficial].

## Background
The method originates from unconstrained isosurface extraction on arbitrary octrees [S1, S3, unofficial]. It leverages the observation that isopolygons obtained in the extraction step form simple closed curves on the surface of convex bodies (octree nodes) [S1, S3, unofficial]. The algorithm is based on a method from [BS95] [S1, S3, unofficial]. It is used in the context of mesh simplification and isosurface extraction from octrees without requiring refinement of the tree or modification of vertex values [S1, S3, unofficial].

## Sources
[S1] Unconstrained isosurface extraction on arbitrary octrees, https://doi.org/10.5555/1281991.1282009, fetched 2026-10-27 (unofficial)
[S2] mkazhdan/IsoSurfaceExtraction, https://github.com/mkazhdan/IsoSurfaceExtraction, fetched 2026-10-27 (unofficial)
[S3] Unconstrained Isosurface Extraction on Arbitrary Octrees, https://hhoppe.com/unconstrainediso.pdf, fetched 2026-10-30 (unofficial)
[S4] Verifiable Visualization for Isosurface Extraction, https://doi.org/10.1109/tvcg.2009.194, fetched 2026-10-26
