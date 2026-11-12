# Plane Representation for Dual Contouring

Also called: Plane Class for Dual Contouring, Plane Constraint Representation, Plane Representation

## Overview
The Plane Representation for Dual Contouring is a geometric data structure used to define linear constraints for locating feature points in dual contouring algorithms [S2, unofficial]. It represents a plane via a unit normal vector and a scalar distance from the origin, enabling efficient signed distance queries [S2, unofficial]. It differs from the parent [Plane Signed Distance Function](1102-plane-signed-distance-function.md) by explicitly exposing the distance parameter and shortest distance vector calculation as part of its interface for iterative constraint solving [S1, S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A point $\mathbf{p} \in \mathbb{R}^3$ and an unnormalized normal vector $\mathbf{n}_{\text{in}} \in \mathbb{R}^3$ for construction; a query point $\mathbf{q} \in \mathbb{R}^3$ for distance evaluation [S1, S2, unofficial].
*   **Output:** A signed distance scalar and a shortest distance vector for a given query point [S1, S2, unofficial].

**Data Structures**
The representation consists of a single class or struct with two fields:
*   `normal`: A `Vector3` representing the unit normal $\mathbf{n}$, guaranteed to be normalized ($\|\mathbf{n}\| = 1$) [S1, S2, unofficial].
*   `distance`: A `float` representing the scalar $d$ in the plane equation $\mathbf{n} \cdot \mathbf{x} + d = 0$ [S1, S2, unofficial].

**Numbered Steps**
1.  **Construction from Point and Normal:**
    a. Normalize the input normal: $\mathbf{n} = \frac{\mathbf{n}_{\text{in}}}{\|\mathbf{n}_{\text{in}}\|}$ [S1, S2, unofficial].
    b. Calculate the distance parameter using the dot product: $d = -\mathbf{n} \cdot \mathbf{p}$ [S1, S2, unofficial].
    c. Store $\mathbf{n}$ and $d$ in the structure [S1, S2, unofficial].
2.  **Signed Distance Query:**
    a. Compute the signed distance: $d_{\text{signed}} = \mathbf{n} \cdot \mathbf{q} + d$ [S1, S2, unofficial].
    b. Return $d_{\text{signed}}$ [S1, S2, unofficial].
3.  **Shortest Distance Vector:**
    a. Compute the signed distance $d_{\text{signed}}$ using Step 2 [S1, S2, unofficial].
    b. Compute the vector from the point to the plane: $\mathbf{v} = d_{\text{signed}} \cdot \mathbf{n}$ [S1, S2, unofficial].
    c. Return $\mathbf{v}$ [S1, S2, unofficial].

**Formulas**
*   Plane Equation: $\mathbf{n} \cdot \mathbf{x} + d = 0$ [S1, S2, unofficial].
*   Distance Parameter: $d = -\mathbf{n} \cdot \mathbf{p}$ [S1, S2, unofficial].
*   Signed Distance: $D(\mathbf{q}) = \mathbf{n} \cdot \mathbf{q} + d$ [S1, S2, unofficial].
*   Shortest Vector: $\mathbf{v} = D(\mathbf{q}) \mathbf{n}$ [S1, S2, unofficial].

**Parameters**
*   Normalization: The normal vector must be normalized before storing [S1, S2, unofficial].
*   Iterative Solving Context: In dual contouring, these planes are used to iteratively move a feature point. A typical step size factor is ~0.7 along the resultant average vector to stabilize convergence [S1, S2, unofficial].

**Edge Cases and Pitfalls**
*   **Zero Normal:** Not given in the sources.
*   **Precision:** Not given in the sources.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Plane:
    def __init__(self, point: Vector3, normal: Vector3):
        self.normal = normal.normalize()
        self.distance = -self.normal.dot(point)

    def signed_distance(self, point: Vector3) -> float:
        return self.normal.dot(point) + self.distance

    def shortest_distance_vector(self, point: Vector3) -> Vector3:
        dist = self.signed_distance(point)
        return self.normal * dist
```

## Subtopics

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Not given in the sources.
*   **Limitations:** Not given in the sources.
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
This representation originates from practical implementations of dual contouring for real-time graphics, specifically discussed in the context of finding feature points by averaging vectors to intersecting planes [S1, S2, unofficial]. It is commonly used in voxel-based mesh generation where planar constraints are derived from surface normals at grid edges [S1, S2, unofficial].

## Sources
[S1] Dual Contouring - Finding the feature point, normals off, https://gamedev.stackexchange.com/questions/111387, fetched 2026-10-28 (unofficial)
[S2] Dual Contouring - Finding the feature point, normals off, https://gamedev.stackexchange.com/questions/111387/dual-contouring-finding-the-feature-point-normals-off?noredirect=1&lq=1, fetched 2026-10-29 (unofficial)
