# Woodcock Tracking with Macrocell Majorant

Also called: Woodcock tracker, stochastic ray marching with majorant

## Overview
Woodcock Tracking with Macrocell Majorant is a stochastic ray marching technique for volume rendering on unstructured tetrahedral meshes that utilizes a coarse mesh hierarchy to compute a tighter majorant extinction value [S1, unofficial]. It solves the problem of high null-collision rates in heterogeneous volumes by bounding the local extinction with a precomputed macrocell value, thereby reducing the number of rejected samples compared to using a global or loose majorant on most datasets [S1, unofficial]. It differs from standard Woodcock tracking by integrating coarse mesh traversal logic to manage the majorant bounds, and from [Hierarchical Trilinear Ray Casting](851-hierarchical-trilinear-ray-casting.md) by relying on stochastic free-path sampling rather than deterministic grid traversal [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, maximum traversal distance $t_{\text{max}}$, current coarse element ID (`coarseTetID`), macrocell majorant value $M_{\text{macro}}$, and a transfer function $\text{tf}$ mapping scalar values to color and opacity [S1, unofficial].
*   **Output:** A color value if a collision occurs within $t_{\text{max}}$, or `false` (transparent) if the ray exits the volume or exceeds $t_{\text{max}}$ [S1, unofficial].

**Data Structures:**
*   **Macrocell Majorant ($M_{\text{macro}}$):** A scalar value representing the upper bound of extinction for a coarse mesh element [S1, unofficial].
*   **Tetrahedron Vertices (`tetVertices`):** An array of four 3D coordinates defining the current tetrahedron [S1, unofficial].
*   **Sample Point (`samplePoint`):** The 3D position in space where the scalar value is sampled [S1, unofficial].

**Parameters:**
*   $t_{\text{max}}$: The maximum distance the ray is allowed to travel [S1, unofficial].

**Algorithm Steps:**
1.  **Initialize Majorant:** Compute the effective majorant $M$ as the product of the transfer function's maximum opacity and the macrocell majorant: $M = \text{tf.opacity} \times M_{\text{macro}}$ [S1, unofficial].
2.  **Check Validity:** If $M \le 0$, the ray is skipped and `false` is returned [S1, unofficial].
3.  **Set Exit Boundary:** Initialize the exit distance for the current coarse element to the smallest representable float less than the current ray distance $t$ to ensure the first sample is processed: $t_{\text{exit}} = \text{nextafter}(t, -\infty)$ [S1, unofficial].
4.  **Initialize State:** Set `sampleTetID` to 0 and `tetVertices` to an empty set [S1, unofficial].
5.  **Stochastic Step:** Generate a random number $r \in [0, 1)$. Advance the ray distance $t$ using the Woodcock step formula:
    $$ t \leftarrow t - \frac{\ln(1 - r)}{M} $$
    [S1, unofficial].
6.  **Check Max Distance:** If $t \ge t_{\text{max}}$, return `false` [S1, unofficial].
7.  **Calculate Sample Point:** Compute the 3D position:
    $$ \mathbf{p} = \mathbf{o} + t \cdot \mathbf{d} $$
    [S1, unofficial].
8.  **Check Element Boundary:** If $t > t_{\text{exit}}$, the ray has crossed a coarse element boundary. Traverse the coarse mesh to find the new tetrahedron containing $\mathbf{p}$:
    $$ (\text{sampleTetID}, \text{tetVertices}) \leftarrow \text{traverse}(\mathbf{p}, \text{coarseTetID}) $$
    [S1, unofficial].
    Update the exit distance for the new tetrahedron by intersecting the ray with the tetrahedron faces:
    $$ t_{\text{exit}} \leftarrow \text{intersectFaces}(t, \mathbf{p}, \mathbf{d}, \text{tetVertices}) $$
    [S1, unofficial].
9.  **Sample Data:** Sample the scalar value $s$ at $\mathbf{p}$ using the current tetrahedron:
    $$ s \leftarrow \text{sampleData}(\mathbf{p}, \text{sampleTetID}, \text{tetVertices}) $$
    [S1, unofficial].
10. **Check Transfer Function Range:** If $s$ is not within the transfer function's valid range, continue to the next iteration [S1, unofficial].
11. **Get Color:** Retrieve the color $C$ and opacity $\alpha$ from the transfer function:
    $$ (C, \alpha) \leftarrow \text{tf.getColor}(s) $$
    [S1, unofficial].
12. **Acceptance Test:** Perform the analog acceptance test. Generate a new random number $r' \in [0, 1)$. If:
    $$ \alpha \times \text{tf.opacity} \ge r' \times M $$
    return $C$ [S1, unofficial]. Otherwise, continue to the next iteration [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def woodcock_tracking_with_macrocell(ray_origin, ray_direction, t_max, coarse_tet_id, macrocell_majorant, tf):
    # 1. Initialize majorant
    majorant = tf.opacity * macrocell_majorant
    if majorant <= 0:
        return False

    # 2. Initialize distance along ray
    t = 0.0
    # 3. Set exit boundary
    t_exit = math.nextafter(t, -float('inf'))

    # 4. Initialize state
    sample_tet_id = 0
    tet_vertices = []

    while True:
        # 5. Stochastic step
        r = random.random()
        t -= math.log(1.0 - r) / majorant

        # 6. Check max distance
        if t >= t_max:
            return False

        # 7. Calculate sample point
        sample_point = [ray_origin[i] + t * ray_direction[i] for i in range(3)]

        # 8. Check element boundary
        if t > t_exit:
            sample_tet_id, tet_vertices = traverse(sample_point, coarse_tet_id)
            t_exit = intersect_faces(t, sample_point, ray_direction, tet_vertices)

        # 9. Sample data
        scalar = sample_data(sample_point, sample_tet_id, tet_vertices)

        # 10. Check transfer function range
        if not tf.in_range(scalar):
            continue

        # 11. Get color
        color = tf.get_color(scalar)

        # 12. Acceptance test
        r_prime = random.random()
        if color.opacity * tf.opacity >= r_prime * majorant:
            return color

def traverse(point, coarse_tet_id):
    # Standard coarse mesh traversal to find the tetrahedron containing point
    # Returns (tet_id, vertices) [G]
    return coarse_tet_id, get_tet_vertices(coarse_tet_id)

def intersect_faces(t, p, d, vertices):
    # Computes the exit distance of ray R(s) = p + s*d from the tetrahedron
    # defined by vertices. Returns the smallest positive s > 0 added to t.
    min_exit = float('inf')
    faces = [(0,1,2), (0,1,3), (0,2,3), (1,2,3)]
    for i, j, k in faces:
        v0, v1, v2 = vertices[i], vertices[j], vertices[k]
        n = cross(sub(v1, v0), sub(v2, v0))
        denom = dot(n, d)
        if abs(denom) > 1e-9:
            dist = dot(n, sub(v0, p)) / denom
            if dist > 0:
                min_exit = min(min_exit, dist)
    return t + min_exit

def cross(a, b): return [a[1]*b[2]-a[2]*b[1], a[2]*b[0]-a[0]*b[2], a[0]*b[1]-a[1]*b[0]]
def dot(a, b): return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]
def sub(a, b): return [a[0]-b[0], a[1]-b[1], a[2]-b[2]]
def sample_data(point, tet_id, vertices): return 0.0 # Standard point-in-tetrahedron scalar lookup [G]
```

**Edge Cases and Pitfalls:**
*   **Overstepping:** When using macrocell values, if the ray crosses a boundary into a denser region, the majorant may no longer be valid. The algorithm must restart traversal from the beginning of the next element to avoid missing collisions [S1, unofficial].
*   **Performance Degradation:** On datasets with shallow trees or specific structures (e.g., Agulhas), macrocell optimizations can decrease performance due to the overhead of restarting element processing [S1, unofficial].

## Subtopics

### [Woodcock Tracking with Coarse Mesh and Macrocells](1048-woodcock-tracking-with-coarse-mesh-and-macrocells.md)
Extends this method by integrating coarse mesh traversal for AMR data, using macrocell densities to provide tighter majorants than transfer function opacity alone.

### [Weighted Decomposition Tracking](9232-weighted-decomposition-tracking.md)
An alternative acceleration technique that decomposes the medium into homogeneous and heterogeneous components, allowing non-bounding control extinction.

## Operating characteristics
*   **Rendering Performance:** Tighter majorants reduce the number of null collisions, improving rendering times on most datasets [S1, unofficial].
*   **Memory Complexity:** Low, as it primarily stores the coarse mesh structure and transfer function data [S1, unofficial].
*   **Measured Performance:** On the Agulhas dataset, macrocell optimizations degraded performance due to traversal overhead [S1, unofficial]. On other datasets, tighter majorants generally improved rendering times [S1, unofficial].
*   **Strengths:** Reduces null collisions in heterogeneous volumes by providing tighter bounds than global majorants [S1, unofficial].
*   **Limitations:** Performance can degrade on specific dataset structures (e.g., shallow trees) due to the cost of boundary traversal and restart logic [S1, unofficial].

## Background
This method is described in the context of direct volume rendering of tree-based tetrahedral adaptive mesh refinement (AMR) data [S1, unofficial]. It builds upon the principles of Woodcock tracking (null-collision method) and adapts them for unstructured tetrahedral meshes using a coarse mesh hierarchy to manage majorant values [S1, unofficial].

## Sources
[S1] Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data, https://repository.bilkent.edu.tr/items/fcbc592a-2646-49af-8b84-67308b6fd3e9, fetched 2026-10-27 (unofficial)
