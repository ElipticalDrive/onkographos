# Barycentric Interpolation on Tetrahedra
Also called: 3D Barycentric Coordinates Interpolation

## Overview
Barycentric interpolation on tetrahedra computes the value of a scalar field at an arbitrary point inside a tetrahedron by linearly interpolating the values at its four vertices using 3D barycentric coordinates [S1, unofficial]. It extends the 2D barycentric interpolation used for triangles to 3D volumes by relating four tetrahedral subvolumes to the total volume [S1, unofficial]. This method serves as the fundamental sampling primitive for tetrahedral volume rendering and mesh processing, providing linearly interpolated values that vary smoothly across the tetrahedral element [S1, unofficial]. It is the 3D simplex counterpart to [Barycentric Interpolation on Triangles](5671-barycentric-interpolation-on-triangles.md), which handles 2D surfaces [S1, unofficial].

## How it works
**Inputs:**
- A tetrahedron defined by four vertices $V_a, V_b, V_c, V_d$ with positions $\mathbf{p}_a, \mathbf{p}_b, \mathbf{p}_c, \mathbf{p}_d \in \mathbb{R}^3$.
- Scalar values $f(V_a), f(V_b), f(V_c), f(V_d)$ associated with each vertex.
- A query point $\mathbf{q} = (x, y, z) \in \mathbb{R}^3$.

**Outputs:**
- The interpolated scalar value $f(\mathbf{q})$.
- A boolean indicating whether $\mathbf{q}$ lies inside the tetrahedron.

**Data Structures:**
- Vertex positions: Array of 4 vectors, each 3 floats.
- Vertex values: Array of 4 floats.
- Barycentric coordinates: Four floats $(\alpha, \beta, \gamma, \delta)$ such that $\alpha + \beta + \gamma + \delta = 1$.

**Steps:**
1. **Compute Barycentric Coordinates:**
   Calculate the barycentric coordinates $(\alpha, \beta, \gamma, \delta)$ of point $\mathbf{q}$ with respect to the tetrahedron vertices. These coordinates represent the normalized volumes of the four sub-tetrahedra formed by $\mathbf{q}$ and the faces of the original tetrahedron.
   - Let $V_{total}$ be the volume of the tetrahedron defined by $V_a, V_b, V_c, V_d$.
   - Let $V_{qbc d}$ be the volume of the tetrahedron formed by $\mathbf{q}, V_b, V_c, V_d$.
   - Let $V_{qac d}$ be the volume of the tetrahedron formed by $\mathbf{q}, V_a, V_c, V_d$.
   - Let $V_{qab d}$ be the volume of the tetrahedron formed by $\mathbf{q}, V_a, V_b, V_d$.
   - Let $V_{qab c}$ be the volume of the tetrahedron formed by $\mathbf{q}, V_a, V_b, V_c$.
   - The coordinates are defined as:
     $$ \alpha = \frac{V_{qbc d}}{V_{total}}, \quad \beta = \frac{V_{qac d}}{V_{total}}, \quad \gamma = \frac{V_{qab d}}{V_{total}}, \quad \delta = \frac{V_{qab c}}{V_{total}} $$
   - Note: The source [S1, unofficial] uses the notation $V_{qbcd}$ etc. to denote the partial volumes. The coefficient for $f(V_a)$ corresponds to the volume opposite to vertex $a$ (i.e., formed by $\mathbf{q}$ and the face $bcd$), which is $V_{qbc d}$. Thus, $\alpha$ is the weight for $f(V_a)$.

2. **Check Containment:**
   The point $\mathbf{q}$ is inside the tetrahedron if and only if all barycentric coordinates are non-negative and their sum is 1 (which is guaranteed by the volume ratio definition if the point is coplanar with the vertices, but numerically we check the bounds). Specifically, the condition is:
   $$ \beta > 0, \quad \gamma > 0, \quad \delta > 0, \quad \text{and} \quad \beta + \gamma + \delta < 1 $$
   Since $\alpha = 1 - (\beta + \gamma + \delta)$, this is equivalent to $\alpha > 0, \beta > 0, \gamma > 0, \delta > 0$ [S1, unofficial].

3. **Interpolate Value:**
   If the point is inside, compute the interpolated value using the linear combination of vertex values weighted by the barycentric coordinates:
   $$ f(\mathbf{q}) = \alpha f(V_a) + \beta f(V_b) + \gamma f(V_c) + \delta f(V_d) $$
   Substituting $\alpha = 1 - \beta - \gamma - \delta$, this matches the formula in [S1, unofficial]:
   $$ f(\mathbf{q}) = (1 - \beta - \gamma - \delta) f(V_a) + \beta f(V_b) + \gamma f(V_c) + \delta f(V_d) $$
   Note: The source [S1, unofficial] uses $V_{qbcd}$ to denote the vertex opposite to the sub-volume $V_{qbc d}$, which is $V_a$. The formula in [S1, unofficial] is:
   $$ f(p) = (1 - \beta - \gamma - \delta) f(V_{qbcd}) + \beta f(V_{qacd}) + \gamma f(V_{qabd}) + \delta f(V_{qabc}) $$
   Here, $V_{qbcd}$ refers to the vertex $a$ (as it is the vertex not in the sub-volume $q,b,c,d$). To avoid confusion, we map:
   - $f(V_{qbcd}) \rightarrow f(V_a)$
   - $f(V_{qacd}) \rightarrow f(V_b)$
   - $f(V_{qabd}) \rightarrow f(V_c)$
   - $f(V_{qabc}) \rightarrow f(V_d)$
   This mapping assumes the standard convention where the barycentric coordinate $\beta$ corresponds to the volume opposite to vertex $b$ (i.e., $q,a,c,d$). However, the source [S1, unofficial] explicitly states:
   $$ f(p) = (1 - \beta - \gamma - \delta) * f(V_{qbcd}) + \beta * f(V_{qacd}) + \gamma * f(V_{qabd}) + \delta * f(V_{qabc}) $$
   And defines $V_{qbcd}$ as a partial volume defined by vertices $a,b,c,d$ and point $q$. The text says "four tetrahedral subvolumes are related to each other". The standard definition is that the weight for vertex $V_i$ is the volume of the sub-tetrahedron formed by the query point and the face opposite to $V_i$.
   - Weight for $V_a$: Volume of $q,b,c,d$. Let's call this $V_a'$.
   - Weight for $V_b$: Volume of $q,a,c,d$. Let's call this $V_b'$.
   - Weight for $V_c$: Volume of $q,a,b,d$. Let's call this $V_c'$.
   - Weight for $V_d$: Volume of $q,a,b,c$. Let's call this $V_d'$.
   The source formula uses $\beta$ for $V_{qacd}$ (which is $V_b'$), $\gamma$ for $V_{qabd}$ (which is $V_c'$), and $\delta$ for $V_{qabc}$ (which is $V_d'$). The term $(1-\beta-\gamma-\delta)$ is therefore the weight for $V_{qbcd}$ (which is $V_a'$).
   So, if we define:
   - $\alpha = 1 - \beta - \gamma - \delta$
   - Then $f(\mathbf{q}) = \alpha f(V_a) + \beta f(V_b) + \gamma f(V_c) + \delta f(V_d)$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def barycentric_interpolation_tetrahedron(vertices, values, q):
    """
    Interpolate scalar value at point q inside a tetrahedron.
    
    Args:
        vertices: List of 4 tuples (x, y, z) for V_a, V_b, V_c, V_d.
        values: List of 4 floats for f(V_a), f(V_b), f(V_c), f(V_d).
        q: Tuple (x, y, z) for the query point.
        
    Returns:
        (interpolated_value, is_inside)
    """
    # Unpack vertices
    Va, Vb, Vc, Vd = vertices
    fa, fb, fc, fd = values
    
    # Compute volumes of sub-tetrahedra
    # Volume of tetrahedron defined by points p1, p2, p3, p4 is 1/6 * |det(p2-p1, p3-p1, p4-p1)|
    def tet_volume(p1, p2, p3, p4):
        v1 = (p2[0]-p1[0], p2[1]-p1[1], p2[2]-p1[2])
        v2 = (p3[0]-p1[0], p3[1]-p1[1], p3[2]-p1[2])
        v3 = (p4[0]-p1[0], p4[1]-p1[1], p4[2]-p1[2])
        det = v1[0]*(v2[1]*v3[2]-v2[2]*v3[1]) - v1[1]*(v2[0]*v3[2]-v2[2]*v3[0]) + v1[2]*(v2[0]*v3[1]-v2[1]*v3[0])
        return abs(det) / 6.0

    # Total volume
    V_total = tet_volume(Va, Vb, Vc, Vd)
    
    if V_total == 0:
        # Degenerate tetrahedron
        return 0.0, False

    # Partial volumes
    # V_qbcd: volume formed by q, Vb, Vc, Vd (opposite to Va)
    V_qbcd = tet_volume(q, Vb, Vc, Vd)
    # V_qacd: volume formed by q, Va, Vc, Vd (opposite to Vb)
    V_qacd = tet_volume(q, Va, Vc, Vd)
    # V_qabd: volume formed by q, Va, Vb, Vd (opposite to Vc)
    V_qabd = tet_volume(q, Va, Vb, Vd)
    # V_qabc: volume formed by q, Va, Vb, Vc (opposite to Vd)
    V_qabc = tet_volume(q, Va, Vb, Vc)

    # Barycentric coordinates
    # alpha corresponds to Va, beta to Vb, gamma to Vc, delta to Vd
    alpha = V_qbcd / V_total
    beta = V_qacd / V_total
    gamma = V_qabd / V_total
    delta = V_qabc / V_total

    # Check if inside
    # Condition from [S1, unofficial]: beta > 0, gamma > 0, delta > 0, and beta + gamma + delta < 1
    # This implies alpha > 0 as well since alpha = 1 - (beta + gamma + delta)
    is_inside = (beta > 0) and (gamma > 0) and (delta > 0) and (beta + gamma + delta < 1)

    if is_inside:
        # Interpolate
        # Formula from [S1, unofficial]:
        # f(p) = (1 - beta - gamma - delta) * f(V_qbcd) + beta * f(V_qacd) + gamma * f(V_qabd) + delta * f(V_qabc)
        # Mapping: V_qbcd -> Va, V_qacd -> Vb, V_qabd -> Vc, V_qabc -> Vd
        f_q = (1 - beta - gamma - delta) * fa + beta * fb + gamma * fc + delta * fd
        return f_q, True
    else:
        return 0.0, False
```

**Parameters:**
- No specific parameters are required for the basic interpolation method.
- Precision: Floating-point precision issues may arise for degenerate tetrahedra (volume near zero) or points very close to the boundary. [G] A small epsilon tolerance should be used for boundary checks.

**Edge Cases and Pitfalls:**
- **Degenerate Tetrahedra:** If the volume is zero, the barycentric coordinates are undefined. The method should handle this by returning a default value or raising an error.
- **Boundary Conditions:** Points on the boundary of the tetrahedron have one or more barycentric coordinates equal to zero. The containment check should be robust to floating-point errors.
- **Outside Points:** If the point is outside, the barycentric coordinates will have at least one negative value. The method returns a flag indicating the point is not inside.

## Subtopics

### [Barycentric Interpolation on Triangles](5671-barycentric-interpolation-on-triangles.md)
Barycentric interpolation on triangles is the method for computing vertex attributes (such as colors, normals, or UV coordinates) at an arbitrary point within a triangle, given the vertex attributes and the point's barycentric coordinates. It serves as the fundamental sampling primitive for triangle-based rendering and mesh processing, providing linearly interpolated values that vary smoothly across the triangle surface. It is the 2D simplex counterpart to [Barycentric Interpolation on Tetrahedra](1843-barycentric-interpolation-on-tetrahedra.md), which extends this concept to 3D volumes using four vertices.

### SDF Aggregation via Barycentric Interpolation
Aggregates new point positions and their SDF values into the existing SDF field using barycentric interpolation within tetrahedra [S2, unofficial]. The method involves cloning SDF values for non-neighbor points, computing inside tetrahedra and barycentric coordinates for updated points, creating a mask for points inside tetrahedra, and updating SDF at mask positions using einsum: `barycentric_coords[mask] dot sdf[tets[inside_tets[mask]]]` [S2, unofficial]. Feature interpolation is optional and follows the same pattern if features are present [S2, unofficial]. Non-neighbor points have their SDF and features set to 0 [S2, unofficial]. Positions are updated for non-neighbor points [S2, unofficial]. This method is used in the TetWeave algorithm [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per query point, assuming the tetrahedron is known. Computing barycentric coordinates involves a constant number of arithmetic operations and determinant calculations.
- **Memory Complexity:** $O(1)$ auxiliary space per query.
- **Measured Performance:** Not given in the sources.
- **Scaling:** The method scales linearly with the number of query points, as each point is processed independently.
- **Strengths:** Simple to implement, computationally efficient, provides smooth linear interpolation within the tetrahedron.
- **Limitations:** Only provides linear interpolation, which may not capture complex field variations. Requires the point to be inside the tetrahedron for valid interpolation. Degenerate tetrahedra can cause numerical instability.
- **When to use:** When linear interpolation within tetrahedral elements is sufficient, such as in basic volume rendering or mesh processing.
- **When not to use:** When higher-order interpolation is required, or when the field varies significantly within the tetrahedron.

## Background
The method extends the well-known 2D barycentric coordinates to 3D by relating four tetrahedral subvolumes [S1, unofficial]. The 2D barycentric coordinates are referenced to [Shirley05] [S1, unofficial]. The approach is discussed in the context of volume rendering techniques, specifically as an alternative to deriving cubic polynomials for tetrahedral interpolation [S1, unofficial]. The method is used in the TetWeave algorithm for SDF aggregation [S2, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] AlexandreBinninger/TetWeave, https://github.com/AlexandreBinninger/TetWeave, fetched 2026-10-27 (unofficial)
