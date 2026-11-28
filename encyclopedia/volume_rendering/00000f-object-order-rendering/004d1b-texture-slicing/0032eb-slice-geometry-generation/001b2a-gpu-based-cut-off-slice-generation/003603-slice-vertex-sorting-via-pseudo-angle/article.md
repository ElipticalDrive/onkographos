# Slice Vertex Sorting via Pseudo-Angle

Also called: Pseudo-Angle Metric Sorting, Pseudo-Angle Sorting

## Overview
This method sorts the intersection vertices of a volume slice around their centroid in counter-clockwise order using a custom pseudo-angle metric [S1, unofficial]. It solves the problem of generating a valid triangle fan or polygon from unordered intersection points, which is a necessary step after [GPU-based Cut-off Slice Generation](948-gpu-based-cut-off-slice-generation.md) produces the raw vertex data [G]. It differs from [Incremental Intersection and Texture Coordinates](997-incremental-intersection-and-texture-coordinates.md) and [View-Aligned Slicing (Proxy Geometry Generation)](7620-view-aligned-slicing-proxy-geometry-generation.md) by relying on a post-hoc geometric sort of the entire vertex set rather than incremental traversal or proxy geometry alignment [S1, unofficial].

## How it works
**Inputs:** A list of 3D vertices $V = \{v_0, v_1, \dots, v_{n-1}\}$ representing the intersection polygon of a slice with the volume bounding box [G].
**Outputs:** A sorted list of vertices $V_{sorted}$ ordered counter-clockwise around the centroid, ready for triangle fan generation [S1, unofficial].

**Data Structures:**
- `vertices`: A dynamic array or vector of 3D points [G].
- `angles`: A parallel array or struct array storing the computed pseudo-angle for each vertex [G].

**Steps:**
1. **Compute Centroid:** Calculate the centroid $C$ by averaging all vertex coordinates: $C = \frac{1}{n} \sum_{i=0}^{n-1} v_i$ [S1, unofficial].
2. **Compute Pseudo-Angle:** For each vertex $v_i$, calculate the delta vector $\Delta = v_i - C$ [S1, unofficial]. Compute the pseudo-angle $A(\Delta)$ using the formula below [S1, unofficial].
3. **Sort:** Sort the vertices based on their computed pseudo-angles in ascending order [S1, unofficial].
4. **Generate Geometry:** If $n > 3$, generate triangles by iterating $i$ from $0$ to $n-1$, setting $j = (i+1)\%n$, and pushing triangles formed by $v_{sorted}[i]$, $C$, and $v_{sorted}[j]$ [S1, unofficial]. If $n \le 3$, push the vertices in sorted order directly [S1, unofficial].

**Formula:**
The pseudo-angle metric $A(p)$ for a delta vector $p = (\Delta x, \Delta y)$ is defined as [S1, unofficial]:

$$
A(p) = \begin{cases} 
-1 & \text{if } \Delta x = 0 \text{ and } \Delta y = 0 \\
2.0 - \frac{\Delta y}{|\Delta x| + |\Delta y|} & \text{if } \Delta x < 0 \\
4.0 + \frac{\Delta y}{|\Delta x| + |\Delta y|} & \text{if } \Delta x \ge 0 
\end{cases}
$$

Where $\Delta x$ and $\Delta y$ are the x and y components of the vector $p = v_i - C$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_pseudo_angle(delta_x, delta_y):
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
    if not vertices:
        return []
    n = len(vertices)
    centroid_x = sum(v.x for v in vertices) / n
    centroid_y = sum(v.y for v in vertices) / n
    angle_list = []
    for i, v in enumerate(vertices):
        dx = v.x - centroid_x
        dy = v.y - centroid_y
        angle = compute_pseudo_angle(dx, dy)
        angle_list.append((angle, i))
    angle_list.sort(key=lambda x: x[0])
    return [vertices[idx] for _, idx in angle_list]

def generate_slice_geometry(sorted_vertices, centroid):
    n = len(sorted_vertices)
    if n > 3:
        triangles = []
        for i in range(n):
            j = (i + 1) % n
            triangles.append((sorted_vertices[i], centroid, sorted_vertices[j]))
        return triangles
    else:
        return sorted_vertices
```

**Parameters:** No tunable parameters. The metric is deterministic [G].

**Edge Cases and Pitfalls:**
- **Centroid Coincidence:** If a vertex lies exactly on the centroid, $\Delta x = 0$ and $\Delta y = 0$, returning $-1$ [S1, unofficial].
- **Precision and Projection:** Not given in the sources.

## Subtopics

### [Dynamic Slice Generation](18041-dynamic-slice-generation.md)
Generates slice geometry by dynamically managing active edges crossing slice boundaries using a priority queue. Chosen when adapting the active edge set to the current view direction and slice spacing in real-time is required, differing from vertex-sorting methods by relying on edge traversal rather than post-hoc vertex ordering.

### [Convex Polygon Point Sorting](18324-convex-polygon-point-sorting.md)
Sorts 3D intersection points forming a convex polygon into counter-clockwise order around a pivot point using a view-dependent cross-product metric. Chosen when ensuring topological correctness for volume rendering via convexity assumptions, differing from general polygon sorting by avoiding geometric centroids or local 2D polar angles.

## Operating characteristics
- **Time Complexity:** $O(n \log n)$ due to sorting, where $n$ is the number of intersection vertices. Computing the centroid and angles is $O(n)$ [G].
- **Memory Complexity:** $O(n)$ to store the angles and sorted indices [G].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:** Simple to implement; does not require complex geometric predicates or cross-product calculations for ordering [G].
- **Limitations:** Relies on a 2D projection; if the slice plane is not aligned with the coordinate system used for the pseudo-angle, the sorting will be incorrect [G]. It assumes the intersection polygon is convex or star-shaped with respect to the centroid [G].
- **When to use:** When the number of vertices per slice is small and a simple, robust sorting method is preferred over incremental traversal [G].

## Background
This method is implemented in a specific open-source codebase [S1, unofficial]. It is used as a utility function within the rendering pipeline to prepare slice geometry for rasterization [G].

## Sources
[S1] Conradyen/Slice-Base-Volume-Rendering, https://github.com/Conradyen/Slice-Base-Volume-Rendering, fetched 2026-10-27 (unofficial)
