# Plane Representation for Dual Contouring

Also called: Plane Class for Dual Contouring, Plane Constraint Representation, Plane Representation

## Overview
The Plane Representation for Dual Contouring is a geometric data structure used to define linear constraints for locating feature points in dual contouring algorithms. It represents a plane via a unit normal vector and a scalar distance from the origin, enabling efficient signed distance queries. It differs from the parent [Plane Signed Distance Function](1102-plane-signed-distance-function.md) by explicitly exposing the distance parameter and shortest distance vector calculation as part of its interface for iterative constraint solving [S1, S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A point $\mathbf{p} \in \mathbb{R}^3$ and a normal vector $\mathbf{n}_{in} \in \mathbb{R}^3$ (unnormalized) for construction; a query point $\mathbf{q} \in \mathbb{R}^3$ for distance evaluation.
*   **Output:** A signed distance scalar $d$ and a shortest distance vector $\mathbf{v}$ for a given query point.

**Data Structures**
The representation consists of a single class or struct with two fields:
1.  `normal`: A `Vector3` representing the unit normal $\mathbf{n}$. It is guaranteed to be normalized ($\|\mathbf{n}\| = 1$).
2.  `distance`: A `float` representing the scalar $d$ in the plane equation $\mathbf{n} \cdot \mathbf{x} + d = 0$.

**Numbered Steps**

1.  **Construction from Point and Normal:**
    Given a point $\mathbf{p}$ on the plane and an arbitrary normal vector $\mathbf{n}_{in}$:
    a. Normalize the input normal: $\mathbf{n} = \frac{\mathbf{n}_{in}}{\|\mathbf{n}_{in}\|}$.
    b. Calculate the distance parameter using the dot product: $d = -\mathbf{n} \cdot \mathbf{p}$.
    c. Store $\mathbf{n}$ and $d$ in the structure.

2.  **Signed Distance Query:**
    Given a query point $\mathbf{q}$:
    a. Compute the signed distance: $d_{signed} = \mathbf{n} \cdot \mathbf{q} + d$.
    b. Return $d_{signed}$.

3.  **Shortest Distance Vector:**
    Given a query point $\mathbf{q}$:
    a. Compute the signed distance $d_{signed}$ using Step 2.
    b. Compute the vector from the point to the plane: $\mathbf{v} = d_{signed} \cdot \mathbf{n}$.
    c. Return $\mathbf{v}$.

**Formulas**
*   Plane Equation: $\mathbf{n} \cdot \mathbf{x} + d = 0$
*   Distance Parameter: $d = -\mathbf{n} \cdot \mathbf{p}$
*   Signed Distance: $D(\mathbf{q}) = \mathbf{n} \cdot \mathbf{q} + d$
*   Shortest Vector: $\mathbf{v} = D(\mathbf{q}) \mathbf{n}$

**Parameters**
*   Normalization: The normal vector must be normalized before storing. [S1, S2, unofficial]
*   Iterative Solving Context: In dual contouring, these planes are used to iteratively move a feature point. A typical step size factor is ~0.7 along the resultant average vector to stabilize convergence [S1, S2, unofficial].

**Edge Cases and Pitfalls**
*   **Zero Normal:** If the input normal $\mathbf{n}_{in}$ is zero, normalization fails. The source code assumes valid input vectors [S1, S2, unofficial].
*   **Precision:** Floating-point precision in the dot product and normalization affects the accuracy of the signed distance, which impacts the convergence of the feature point location algorithm [S1, S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Plane:
    def __init__(self, point: Vector3, normal: Vector3):
        # Normalize the input normal vector
        self.normal = normal.normalize()
        # Calculate distance d such that n . x + d = 0
        # d = -n . p
        self.distance = -self.normal.dot(point)

    def signed_distance(self, point: Vector3) -> float:
        # Returns the signed distance from point to the plane
        return self.normal.dot(point) + self.distance

    def shortest_distance_vector(self, point: Vector3) -> Vector3:
        # Returns the vector from the point to its projection on the plane
        dist = self.signed_distance(point)
        return self.normal * dist
```

## Subtopics

## Operating characteristics
*   **Time Complexity:** Construction is $O(1)$ (normalization and dot product). Distance query is $O(1)$.
*   **Memory Complexity:** $O(1)$ per plane (one vector, one scalar).
*   **Strengths:** Simple, efficient representation for linear constraints. Directly supports the iterative averaging method used in dual contouring feature point location.
*   **Limitations:** Represents only flat planes. Cannot directly model curved surfaces without tessellation or higher-order primitives.
*   **When to use:** When implementing dual contouring or other mesh generation techniques that require solving for a point satisfying multiple linear inequality constraints.
*   **When not to use:** For general ray tracing of planes where a different representation might be more efficient, or for non-linear surfaces.

## Background
This representation originates from practical implementations of dual contouring for real-time graphics, specifically discussed in the context of finding feature points by averaging vectors to intersecting planes [S1, S2, unofficial]. It is commonly used in voxel-based mesh generation where planar constraints are derived from surface normals at grid edges.

## Sources
[S1] Dual Contouring - Finding the feature point, normals off, https://gamedev.stackexchange.com/questions/111387, fetched 2026-10-28 (unofficial)
[S2] Dual Contouring - Finding the feature point, normals off, https://gamedev.stackexchange.com/questions/111387/dual-contouring-finding-the-feature-point-normals-off?noredirect=1&lq=1, fetched 2026-10-29 (unofficial)
