# Dual Contouring Feature Point Iterative Projection

Also called: Feature Point Iterative Projection, Iterative Feature Point Projection

## Overview
This method computes the position of a single vertex (the "feature point") inside a voxel cell for Dual Contouring by iteratively projecting a point onto planes defined by intersecting edges and their normals [S1, S2, unofficial]. It solves the problem of finding a vertex that best represents the intersection of multiple surface constraints within a cell [S1, S2, unofficial]. It differs from [Dual Contouring](778-dual-contouring.md) by using a fixed-iteration heuristic rather than solving a constrained least-squares system [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A voxel cell with $k$ intersecting edges. For each intersecting edge $i$, the algorithm requires the edge midpoint $\mathbf{m}_i$ and the surface normal $\mathbf{n}_i$ at that midpoint [S1, S2, unofficial].
*   **Output:** A 3D point $\mathbf{c}$ representing the feature point vertex for the cell [S1, S2, unofficial].

**Data Structures:**
*   **Plane:** Defined by an origin point $\mathbf{o}$ and a normal vector $\mathbf{n}$ [S1, S2, unofficial].
*   **Vector:** 3D floating-point vector $\mathbf{v} = (x, y, z)$ [G].

**Parameters:**
*   **Iterations ($N$):** The number of projection steps. Sources use $N=5$ [S1, S2, unofficial].
*   **Step Size Factor ($\alpha$):** A multiplier for the averaged displacement vector. Sources use $\alpha = 0.7$ [S1, S2, unofficial]. Convergence occurs without scaling, but scaling stabilizes the iteration [S1, S2, unofficial].

**Steps:**
1.  **Initialization:** Set the current point $\mathbf{c}$ to the center of the voxel cell [S1, S2, unofficial].
2.  **Plane Construction:** For each of the $k$ intersecting edges in the cell, create a plane $P_i$ where the origin is the edge midpoint $\mathbf{m}_i$ and the normal is the edge normal $\mathbf{n}_i$ [S1, S2, unofficial].
3.  **Iteration:** Repeat for $i = 0$ to $N-1$:
    a.  **Distance Calculation:** For each plane $P_j$ (where $j$ indexes the intersecting edges), calculate the signed distance from the current point $\mathbf{c}$ to the plane:
        $$ d_j = \mathbf{n}_j \cdot (\mathbf{c} - \mathbf{m}_j) $$
        [S1, S2, unofficial]
    b.  **Displacement Vector:** Calculate the shortest distance vector (displacement) for each plane:
        $$ \mathbf{v}_j = d_j \mathbf{n}_j $$
        [S1, S2, unofficial]
    c.  **Averaging:** Compute the average of all displacement vectors:
        $$ \mathbf{v}_{\text{avg}} = \frac{1}{k} \sum_{j=0}^{k-1} \mathbf{v}_j $$
        [S1, S2, unofficial]
    d.  **Update:** Update the point $\mathbf{c}$ by subtracting the averaged displacement vector scaled by $\alpha$:
        $$ \mathbf{c} \leftarrow \mathbf{c} - \alpha \mathbf{v}_{\text{avg}} $$
        [S1, S2, unofficial]
        *Note:* An equivalent formulation projects the point onto each plane individually and averages the projected points: $\mathbf{c}_{\text{new}} = \frac{1}{k} \sum_{j=0}^{k-1} (\mathbf{c} - d_j \mathbf{n}_j)$ [S1, S2, unofficial]. This is mathematically identical to the update rule above [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_feature_point(intersecting_edges, cell_center):
    """
    Computes the feature point for a voxel cell using iterative projection.
    
    Args:
        intersecting_edges: List of objects/tuples, each containing:
            - mid: Vector3 (edge midpoint)
            - normal: Vector3 (surface normal at midpoint, normalized)
        cell_center: Vector3 (center of the voxel)
        
    Returns:
        Vector3: The computed feature point
    """
    # Self-contained vector definitions
    class Vector3:
        def __init__(self, x, y, z):
            self.x, self.y, self.z = x, y, z
        def __sub__(self, other):
            return Vector3(self.x - other.x, self.y - other.y, self.z - other.z)
        def __add__(self, other):
            return Vector3(self.x + other.x, self.y + other.y, self.z + other.z)
        def __mul__(self, scalar):
            return Vector3(self.x * scalar, self.y * scalar, self.z * scalar)
        def __truediv__(self, scalar):
            return Vector3(self.x / scalar, self.y / scalar, self.z / scalar)
        def __neg__(self):
            return Vector3(-self.x, -self.y, -self.z)

    def dot(a, b):
        return a.x * b.x + a.y * b.y + a.z * b.z

    iterations = 5
    alpha = 0.7
    c = cell_center
    planes = []
    for edge in intersecting_edges:
        planes.append({'origin': edge.mid, 'normal': edge.normal})
    k = len(planes)

    for _ in range(iterations):
        displacement_sum = Vector3(0, 0, 0)
        for plane in planes:
            diff = c - plane['origin']
            distance = dot(plane['normal'], diff)
            displacement = plane['normal'] * distance
            displacement_sum = displacement_sum + displacement
        avg_displacement = displacement_sum / k
        c = c - alpha * avg_displacement
    return c
```

**Edge Cases and Pitfalls:**
*   **Non-intersecting Planes:** The algorithm may fail if the planes defined by the edge normals do not intersect in a common region [S1, S2, unofficial].
*   **Direction of Update:** It is critical to *subtract* the averaged displacement vector; adding it moves the point away from the intersection minimum [S1, S2, unofficial].
*   **Convergence:** While 5 iterations are suggested, the method is a heuristic and convergence occurs without the step size scaling [S1, S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.
*   **Strengths:** Not given in the sources.
*   **Limitations:** Not given in the sources.
*   **When to Use:** Not given in the sources.
*   **When Not to Use:** Not given in the sources.

## Background
This method is discussed in the context of implementing Dual Contouring for voxel-based games [S1, S2, unofficial]. It was shared by community members as a simpler alternative to solving the constrained least-squares system required for optimal Dual Contouring [S1, S2, unofficial]. The specific iterative projection technique with a 0.7 step size was proposed to stabilize convergence [S1, S2, unofficial].

## Sources
[S1] Dual Contouring - Finding the feature point, normals off, https://gamedev.stackexchange.com/questions/111387, fetched 2026-10-28 (unofficial)
[S2] Dual Contouring - Finding the feature point, normals off, https://gamedev.stackexchange.com/questions/111387/dual-contouring-finding-the-feature-point-normals-off?noredirect=1&lq=1, fetched 2026-10-29 (unofficial)
