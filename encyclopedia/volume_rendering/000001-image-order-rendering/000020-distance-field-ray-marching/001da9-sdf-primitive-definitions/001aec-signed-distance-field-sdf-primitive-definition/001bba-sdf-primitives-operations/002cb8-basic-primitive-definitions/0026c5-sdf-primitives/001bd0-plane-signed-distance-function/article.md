# Plane Signed Distance Function

Also called: Plane SDF, Plane distance, Signed Distance Function (SDF) for Plane, Signed Plane, sdPlane

## Overview
The Plane Signed Distance Function (SDF) computes the exact signed Euclidean distance from a query point to an infinite plane defined by a unit normal vector and an offset. It serves as a fundamental building block for constructive solid geometry (CSG) operations, such as clipping or union with other primitives, and provides a ground plane for scene composition in ray marching. Unlike the [Sphere SDF](1237-sphere-signed-distance-function.md) which defines a bounded volume, the plane defines an unbounded half-space boundary, making it computationally cheaper and analytically simpler, though it lacks the curvature information useful for certain blending operations.

## How it works
**Inputs and Outputs:**
*   **Input:** A query point $\mathbf{p} \in \mathbb{R}^3$ and plane parameters defining the plane geometry.
*   **Output:** A scalar $d \in \mathbb{R}$ representing the signed distance. $d < 0$ indicates the point is on the side opposite to the normal (inside/back), $d > 0$ indicates the point is on the side of the normal (outside/front), and $d = 0$ indicates the point lies on the plane.

**Data Structures:**
The plane is typically represented by two fields:
1.  `normal`: A 3D vector $\mathbf{n} = (n_x, n_y, n_z)$ representing the plane's orientation. It **must** be normalized ($\|\mathbf{n}\| = 1$).
2.  `offset` (or `h`): A scalar representing the signed distance from the origin to the plane along the normal.

Alternatively, the plane can be represented by a point $\mathbf{r}$ on the plane and the normal $\mathbf{n}$, or by the 4D homogeneous coefficients $\mathbf{n}_{4} = (n_x, n_y, n_z, n_w)$ where $n_w$ is the offset.

**Formulas:**
The signed distance $d$ is calculated using the dot product of the query point and the normal, adjusted by the offset.

Definition 1 (Normal + Offset):
$$ d = \mathbf{p} \cdot \mathbf{n} + h $$
Where:
*   $\mathbf{p}$ is the query point.
*   $\mathbf{n}$ is the unit normal vector.
*   $h$ is the offset (distance from origin to plane along $\mathbf{n}$). [S1, S3, unofficial]

Definition 2 (Point on Plane + Normal):
If the plane is defined by a point $\mathbf{r}$ on the plane and normal $\mathbf{n}$, the offset $h$ is derived as $h = -\mathbf{r} \cdot \mathbf{n}$. Substituting this into Definition 1:
$$ d = \mathbf{p} \cdot \mathbf{n} - \mathbf{r} \cdot \mathbf{n} = (\mathbf{p} - \mathbf{r}) \cdot \mathbf{n} $$
This matches the form $d = \mathbf{x} \cdot \mathbf{n} - \mathbf{r} \cdot \mathbf{n}$ found in [S6] (where $\mathbf{r}$ is denoted as the intersection point). [S5, unofficial][S6]

Definition 3 (Homogeneous Coefficients):
Using plane equation coefficients $\mathbf{n}_{4} = (n_x, n_y, n_z, n_w)$:
$$ d = \mathbf{n}_{xyz} \cdot \mathbf{p} + n_w $$
Where $\mathbf{n}_{xyz}$ must be normalized. [S2, unofficial]

**Parameters:**
*   $\mathbf{n}$: Unit normal vector.
*   $h$: Offset scalar.
*   $\mathbf{r}$: Point on the plane (alternative parameterization).

**Pitfalls:**
*   **Normalization:** The normal vector $\mathbf{n}$ must be normalized. If $\mathbf{n}$ is not unit length, the result is not a Euclidean distance but a scaled projection. [S2, unofficial][S6]
*   **Sign Convention:** The sign of the distance depends on the direction of the normal. Ensure consistency with the rest of the scene (e.g., inside/outside conventions for CSG operations).
*   **Infinite Extent:** The function represents an infinite plane. It does not account for plane bounds (e.g., rectangular planes) unless combined with other primitives via CSG.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sd_plane(p: Vector3, normal: Vector3, offset: float) -> float:
    """
    Computes the signed distance from point p to a plane defined by normal and offset.
    
    Args:
        p: Query point (3D vector).
        normal: Plane normal vector (must be normalized).
        offset: Signed distance from origin to plane along normal.
        
    Returns:
        Signed distance. Negative if p is on the back side of the plane,
        positive if on the front side.
    """
    # Ensure normal is normalized [G]
    # In practice, assume input normal is normalized as per [S2, unofficial]
    
    # Calculate dot product of point and normal
    dot_product = p.x * normal.x + p.y * normal.y + p.z * normal.z
    
    # Add offset
    distance = dot_product + offset
    
    return distance

def sd_plane_from_point(p: Vector3, point_on_plane: Vector3, normal: Vector3) -> float:
    """
    Computes the signed distance from point p to a plane defined by a point on the plane and normal.
    
    Args:
        p: Query point (3D vector).
        point_on_plane: A known point lying on the plane.
        normal: Plane normal vector (must be normalized).
        
    Returns:
        Signed distance.
    """
    # Calculate vector from point_on_plane to p
    vec = p - point_on_plane
    
    # Dot product with normal
    distance = vec.x * normal.x + vec.y * normal.y + vec.z * normal.z
    
    return distance
```

## Subtopics

### Plane Representation for Dual Contouring
A specialized usage of the plane SDF in dual contouring algorithms to constrain feature point locations. The plane is defined by a normal and a distance from the origin, used to define constraints for optimization.
*   **Construction:** Given a point $\mathbf{p}$ and normal $\mathbf{n}$, the distance is computed as $d = -\mathbf{n} \cdot \mathbf{p}$. The normal must be normalized. [S7, unofficial]
*   **Distance Calculation:** The signed distance from a query point $\mathbf{q}$ to the plane is $d = \mathbf{n} \cdot \mathbf{q} + d_{offset}$, where $d_{offset}$ is the precomputed distance. [S7, unofficial]
*   **Shortest Distance Vector:** The vector from the query point to the plane is $\mathbf{v} = \mathbf{n} \times d(\mathbf{q})$. This vector is used to iteratively adjust feature points toward the surface. [S7, unofficial]

## Operating characteristics
*   **Time Complexity:** $O(1)$. The calculation involves a fixed number of arithmetic operations (3 multiplications, 2 additions for the dot product, plus one addition for the offset).
*   **Memory Complexity:** $O(1)$. Requires storing only the normal (3 floats) and offset (1 float), or a point and normal.
*   **Strengths:** Extremely fast evaluation; analytically exact; no iterative refinement needed; numerically stable if the normal is normalized.
*   **Limitations:** Represents an infinite plane; cannot model bounded planar surfaces directly without CSG; sensitive to normal normalization.
*   **When to use:** As a ground plane, for clipping volumes, or as a component in CSG operations (union, intersection, difference) with other primitives.
*   **When not to use:** When a bounded planar surface is required (use a box or custom mesh); when the normal is not known or difficult to normalize.

## Background
The signed distance function for a plane is a basic primitive in the field of ray marching and implicit surface rendering. It is derived from the standard plane equation in 3D geometry. The formulation $f(\mathbf{x}) = \mathbf{x} \cdot \mathbf{n} - h$ is attributed to Hart in the context of GPU ray marching of distance fields [S5, unofficial]. The GLSL implementation `sdPlane` is widely available in open-source SDF primitive libraries [S2, unofficial]. The use of plane representations in dual contouring for feature point extraction is discussed in game development contexts [S7, unofficial].

## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S2] marklundin/glsl-sdf-primitives, https://github.com/marklundin/glsl-sdf-primitives, fetched 2026-10-26 (unofficial)
[S3] bhubbard/sdf-math-rs, https://github.com/bhubbard/sdf-math-rs, fetched 2026-10-27 (unofficial)
[S4] zichenwang01/relaxed-boundary, https://github.com/zichenwang01/relaxed-boundary, fetched 2026-10-27 (unofficial)
[S5] GPU Ray Marching of Distance Fields, https://www2.compute.dtu.dk/pubdb/edoc/imm6392.pdf, fetched 2026-10-27 (unofficial)
[S6] Sphere tracing: a geometric method for the antialiased ray tracing of implicit surfaces, http://graphics.cs.illinois.edu/sites/default/files/zeno.pdf, fetched 2026-10-29
[S7] Dual Contouring - Finding the feature point, normals off, https://gamedev.stackexchange.com/questions/111387/dual-contouring-finding-the-feature-point-normals-off?noredirect=1&lq=1, fetched 2026-10-29 (unofficial)
