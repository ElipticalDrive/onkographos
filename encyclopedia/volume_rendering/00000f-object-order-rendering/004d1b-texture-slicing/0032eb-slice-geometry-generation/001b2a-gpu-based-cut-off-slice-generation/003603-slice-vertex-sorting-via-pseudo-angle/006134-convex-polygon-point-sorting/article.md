# Convex Polygon Point Sorting

Also called: Angular Sort around Pivot, Cross-Product Angular Sort

## Overview
This method sorts a set of 3D intersection points that form a convex polygon into a counter-clockwise (CCW) order around a pivot point, producing a vertex sequence suitable for rendering (e.g., as a triangle fan or strip) [S1, unofficial]. It differs from general polygon sorting by relying on the convexity of the input and using a view-dependent cross-product metric rather than a geometric centroid or polar angle in a local 2D plane [S1, unofficial]. It is a core step in GPU-based cut-off slice generation, ensuring that the resulting slice geometry is topologically correct for volume rendering [S1, unofficial].

## How it works
**Inputs:**
- `points`: A list of $N$ 3D points ($\mathbb{R}^3$) representing the intersection of a slicing plane with a volume boundary. The points are assumed to form a convex polygon [S1, unofficial].
- `p0`: The pivot point (typically the first point in the list or the centroid, though the source implementation uses the first point as the reference for the cross product) [S1, unofficial].
- `up`: The view direction or normal vector used to define the orientation of the sorting [S1, unofficial].

**Outputs:**
- `sorted_points`: A list of the input points sorted in counter-clockwise order around `p0` relative to the `up` vector [S1, unofficial].

**Data Structures:**
- The points are stored as 3D vectors. No additional bit-packing is specified; standard floating-point vectors are used [S1, unofficial].

**Steps:**
1.  Define a comparator function that takes two points $a$ and $b$ from the input list.
2.  Compute the vectors from the pivot to the points: $\text{va} = a - p_0$ and $\text{vb} = b - p_0$ [S1, unofficial].
3.  Compute the cross product of these vectors: $\text{cross} = \text{va} \times \text{vb}$ [S1, unofficial].
4.  Compute the dot product of the cross product with the `up` vector: $\text{score} = \text{up} \cdot \text{cross}$ [S1, unofficial].
5.  The sort key is determined by the sign of this score. If $\text{up} \cdot (\text{va} \times \text{vb}) \ge 0$, then $a$ precedes $b$ in the sorted order (i.e., $a$ is "to the left" of $b$ relative to the pivot and view direction) [S1, unofficial].
6.  Apply a stable sorting algorithm using this comparator to the entire list of points [S1, unofficial].

**Formula:**
The comparison between two points $a$ and $b$ is defined as:
$$ \text{up} \cdot ((a - p_0) \times (b - p_0)) \ge 0 $$
Where:
- $p_0$ is the pivot point.
- $\text{up}$ is the reference vector (view direction/normal).
- $\cdot$ denotes the dot product.
- $\times$ denotes the cross product.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sort_convex_polygon(points, p0, up):
    """
    Sorts points in CCW order around p0 relative to up.
    
    Args:
        points: List of 3D vectors (list of tuples or arrays).
        p0: 3D vector, the pivot point.
        up: 3D vector, the reference direction.
        
    Returns:
        List of points sorted in CCW order.
    """
    # Helper function for cross product
    def cross(v1, v2):
        return (
            v1[1]*v2[2] - v1[2]*v1[1], # Note: Standard cross product formula
            v1[2]*v2[0] - v1[0]*v2[2],
            v1[0]*v2[1] - v1[1]*v2[0]
        )
    
    # Helper function for dot product
    def dot(v1, v2):
        return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]

    # Comparator function
    def compare(a, b):
        # va = a - p0
        va = (a[0]-p0[0], a[1]-p0[1], a[2]-p0[2])
        # vb = b - p0
        vb = (b[0]-p0[0], b[1]-p0[1], b[2]-p0[2])
        
        # cross = va x vb
        cross_prod = cross(va, vb)
        
        # score = up . cross
        score = dot(up, cross_prod)
        
        # Return True if a should come before b (CCW order)
        # The source uses >= 0 for a < b in sort order
        return score >= 0

    # Stable sort using the comparator
    # Note: Python's sort is stable, but standard sort doesn't take a comparator directly in Python 3.
    # We use functools.cmp_to_key for compatibility with the logic described.
    import functools
    
    def cmp_func(a, b):
        if compare(a, b):
            return -1 # a comes before b
        else:
            return 1  # b comes before a or equal
            
    sorted_points = sorted(points, key=functools.cmp_to_key(cmp_func))
    return sorted_points
```

**Edge Cases and Pitfalls:**
- **Collinear Points:** If points are collinear with $p_0$, the cross product is zero, and the dot product is zero. The comparator returns `True` ($0 \ge 0$), preserving the original relative order due to stable sorting. This is generally acceptable for convex polygons where collinear vertices are rare or degenerate [S1, unofficial].
- **Pivot Selection:** The source implementation uses the first point in the list as $p_0$. If the first point is not part of the convex hull boundary in a way that allows a consistent angular sweep (e.g., if the polygon is not star-shaped with respect to $p_0$), the sort may fail. However, for a convex polygon, any vertex can serve as a pivot for a CCW sort if the view direction is consistent with the polygon's normal [S1, unofficial].
- **View Direction Alignment:** The `up` vector must be consistent with the desired winding order. If `up` is flipped, the sort order reverses to clockwise [S1, unofficial].

## Subtopics

### Convex Hull Triangulation
This variant, described in [S2, unofficial], orders vertices of a convex, monotone, simple polygon (resulting from plane-hexahedron intersection) into a triangle fan. It differs from the angular sort by explicitly constructing an origin outside the polygon and using dot-product-based angle comparisons.
- **Definition:** An algorithm to order vertices of a convex, monotone, simple polygon into a triangle fan [S2, unofficial].
- **Steps:**
    1.  Compute the centroid of the proxy polygon as the average of all points [S2, unofficial].
    2.  Calculate a new origin outside the polygon: take the vector from the world origin to the centroid, add to it the vector from the centroid to one of the points (doubled in length). This new point becomes the origin $v_0$ [S2, unofficial].
    3.  Let $v_1$ be the closest point to $v_0$. Let the remaining points be $v_i$ ($i=2..n$) [S2, unofficial].
    4.  Sort the remaining points $v_i$ by the angle between vector $v_0v_1$ and $v_0v_i$ using the inner product (dot product) [S2, unofficial].
    5.  Tesselate the proxy polygon into triangle fans and add vertices to the output array in the sorted order [S2, unofficial].
- **Pitfall:** The polygon must be simple, monotone, and convex for this specific algorithm to work correctly [S2, unofficial].
- **Complexity:** $O(n^2)$ in its re-ordering stage, where the unit operation is a dot-product (three multiplications) [S2, unofficial].

## Operating characteristics
- **Time Complexity:** The angular sort using a standard comparison-based sort is $O(N \log N)$, where $N$ is the number of vertices [S1, unofficial]. The convex hull triangulation variant is $O(N^2)$ in its re-ordering stage [S2, unofficial].
- **Memory Complexity:** $O(N)$ to store the points and the sorted list [S1, unofficial].
- **Performance:** The angular sort is efficient and suitable for real-time rendering. The convex hull variant is noted as "quite fast" compared to other convex hull algorithms, but its $O(N^2)$ complexity makes it less scalable for very high vertex counts [S2, unofficial].
- **Strengths:** Simple to implement; robust for convex polygons; view-dependent orientation ensures correct winding for rendering [S1, unofficial].
- **Limitations:** Requires convex input; sensitive to the choice of pivot and view direction; the convex hull variant requires the polygon to be simple, monotone, and convex [S1, unofficial][S2, unofficial].
- **When to use:** Use the angular sort for general convex polygon vertex ordering in slice generation. Use the convex hull triangulation variant when specifically constructing triangle fans from a known convex, monotone polygon and $N$ is small [S2, unofficial].

## Background
The angular sort method originates from the SmokeSimulation project, which implements real-time volumetric rendering techniques [S1, unofficial]. The convex hull triangulation algorithm is described in a thesis on real-time volumetric rendering of fire in a production system, which focuses on optimizing slice generation for hexahedral volumes [S2, unofficial]. Both methods are part of the broader field of GPU-based cut-off slice generation for volume rendering.

## Sources
[S1] nevermoe/SmokeSimulation, https://github.com/nevermoe/SmokeSimulation, fetched 2026-10-26 (unofficial)
[S2] PDF Real-Time Volumetric Rendering of Fire in a Production System ..., https://clas.iusb.edu/math-compsci/_prior-thesis/YVanzine_thesis.pdf, fetched 2026-10-27 (unofficial)
