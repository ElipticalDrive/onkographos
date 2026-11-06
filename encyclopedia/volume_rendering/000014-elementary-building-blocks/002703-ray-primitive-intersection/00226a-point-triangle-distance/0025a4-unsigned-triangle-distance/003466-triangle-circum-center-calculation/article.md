# Triangle Circum-Center Calculation

Also called: Circum-center, Circumcircle Center

## Overview
Computes the center and radius of the unique circle passing through three non-collinear vertices $a, b, c$ of a triangle. This method is primarily used in volume rendering to define the bounding sphere (splat) for triangle-based leaf nodes, as described in [Unsigned Triangle Distance](3587-unsigned-triangle-distance.md). It is distinguished by its formulation using vector cross products and coordinate differences, which minimizes numerical error for vertices far from the origin, and by a specific fallback strategy for obtuse triangles.

## How it works
**Inputs:** Three vertices $a, b, c \in \mathbb{R}^3$ defining a triangle.
**Outputs:** The circum-center $C$ and circum-radius $R$.

**Data Structures:**
No persistent data structures are required. The calculation is stateless and computes values directly from the input coordinates.

**Steps:**
1.  **Compute Edge Vectors and Cross Product:**
    Calculate the vectors from vertex $a$ to $b$ and $c$:
    $$ \vec{ab} = b - a $$
    $$ \vec{ac} = c - a $$
    Compute the area vector $A(b, c)$, which is perpendicular to the triangle plane:
    $$ A(b, c) = \vec{ab} \times \vec{ac} $$
    *Note: The source defines $A(u, v) = (u - a) \times (v - a)$ [S1, unofficial].*

2.  **Check for Degeneracy:**
    Calculate the squared magnitude of the area vector:
    $$ |A(b, c)|^2 = A(b, c) \cdot A(b, c) $$
    If $|A(b, c)|^2$ is effectively zero, the vertices are collinear (degenerate triangle). The method is undefined for this case. In practice, degenerate triangles are removed or collapsed prior to this calculation [S1, unofficial].

3.  **Compute Relative Circum-Center ($m_{tri}$):**
    If the triangle is non-degenerate, compute the vector from $a$ to the circum-center:
    $$ m_{tri} = \frac{|c - a|^2 \left( A(b, c) \times (b - a) \right) + |b - a|^2 \left( (c - a) \times A(b, c) \right)}{2 |A(b, c)|^2} $$
    Where:
    *   $|c - a|^2$ is the squared length of edge $ac$.
    *   $|b - a|^2$ is the squared length of edge $ab$.
    *   The numerator involves two cross products.

4.  **Compute Absolute Circum-Center ($C$) and Radius ($R$):**
    $$ C = a + m_{tri} $$
    $$ R = |m_{tri}| $$

5.  **Handle Obtuse Triangles (Fallback):**
    If the triangle is non-degenerate but obtuse, the circum-center may lie far outside the triangle, potentially causing issues for splat-based rendering. The source recommends a simpler approximation for such cases:
    *   Identify the longest edge among $ab, bc, ca$.
    *   Set the circum-center $C$ to the midpoint of this longest edge.
    *   Set the circum-radius $R$ to half the length of this longest edge.
    *   *Note: The source states this approximation is "sufficient" for remaining obtuse triangles after degenerate removal [S1, unofficial].*

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def triangle_circum_center(a, b, c):
    """
    Computes the circum-center and circum-radius of a triangle defined by vertices a, b, c.
    Returns (center, radius).
    """
    # Step 1: Edge vectors
    ab = b - a
    ac = c - a
    
    # Step 2: Area vector and its squared magnitude
    # A(b, c) = (b - a) x (c - a)
    area_vec = cross(ab, ac)
    area_sq = dot(area_vec, area_vec)
    
    # Check for degeneracy
    if area_sq < 1e-12: # [G] Threshold is implementation-dependent; source notes denominator vanishes
        raise ValueError("Degenerate triangle (collinear vertices)")
    
    # Step 3: Relative circum-center m_tri
    # Formula: m_tri = (|c-a|^2 * (A x ab) + |b-a|^2 * (ac x A)) / (2 * |A|^2)
    # Note: Source uses A(b,c) x (b-a) and (c-a) x A(b,c)
    
    term1 = dot(ac, ac) * cross(area_vec, ab)
    term2 = dot(ab, ab) * cross(ac, area_vec)
    
    m_tri = (term1 + term2) / (2.0 * area_sq)
    
    # Step 4: Absolute center and radius
    center = a + m_tri
    radius = length(m_tri)
    
    # Step 5: Check for obtuse triangle and apply fallback if necessary
    # A triangle is obtuse if the circumcenter lies outside.
    # A simple check is if the dot product of any two edge vectors is negative.
    # However, the source implies a heuristic or explicit check. 
    # We check if the center is "far" or use edge lengths to determine obtuseness.
    # Standard geometric test: c^2 > a^2 + b^2 for some permutation.
    
    len_ab_sq = dot(ab, ab)
    len_bc_sq = dot(c - b, c - b)
    len_ca_sq = dot(ac, ac)
    
    is_obtuse = (len_ab_sq > len_bc_sq + len_ca_sq) or \
                (len_bc_sq > len_ab_sq + len_ca_sq) or \
                (len_ca_sq > len_ab_sq + len_bc_sq)
                
    if is_obtuse:
        # Fallback: Midpoint of longest edge
        edges = [
            (length(ab), a, b),
            (length(c - b), b, c),
            (length(ac), a, c)
        ]
        # Find edge with max length
        longest_edge = max(edges, key=lambda x: x[0])
        mid = (longest_edge[1] + longest_edge[2]) / 2.0
        center = mid
        radius = longest_edge[0] / 2.0
        
    return center, radius

def cross(u, v):
    return [
        u[1]*v[2] - u[2]*v[1],
        u[2]*v[0] - u[0]*v[2],
        u[0]*v[1] - u[1]*v[0]
    ]

def dot(u, v):
    return u[0]*v[0] + u[1]*v[1] + u[2]*v[2]

def length(v):
    return math.sqrt(dot(v, v))
```

**Parameters:**
*   Numerical tolerance for degeneracy check: Not given in the sources. [G] A small epsilon (e.g., $10^{-12}$) is typically used.

**Edge Cases and Pitfalls:**
*   **Degeneracy:** The denominator $2 |A(b, c)|^2$ vanishes when vertices are collinear. The method is unstable here. Degenerate triangles must be handled before this calculation [S1, unofficial].
*   **Obtuse Triangles:** While the formula works mathematically, the resulting circum-center can be far from the triangle, leading to large bounding spheres. The source recommends the midpoint-of-longest-edge fallback for obtuse triangles [S1, unofficial].
*   **Numerical Precision:** The formula uses differences of coordinates ($b-a$, etc.), which reduces relative error compared to using absolute coordinates, as vertices are usually closer to each other than to the origin [S1, unofficial].

## Subtopics

### Tetrahedron Circum-Center Calculation
Computes the circum-center and radius for a tetrahedron defined by vertices $a, b, c, d$. It extends the triangle method to 3D. The relative circum-center $m_{tet}$ is given by:
$$ m_{tet} = \frac{|d - a|^2 A(b, c) + |c - a|^2 A(d, b) + |b - a|^2 A(c, d)}{2 \det(M)} $$
where $A(u, v) = (u - a) \times (v - a)$ and the denominator is $2$ times the determinant of the matrix formed by the components of vectors $b-a, c-a, d-a$. Specifically, the source shows the denominator as a product of components in a specific notation, but standard interpretation is the scalar triple product or determinant. The absolute center is $a + m_{tet}$ and radius is $|m_{tet}|$. The pitfall is that the denominator vanishes if the tetrahedron is degenerate (flat or needle-shaped). Using coordinate differences reduces relative error [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$. The calculation involves a fixed number of vector operations (cross products, dot products, additions, multiplications).
*   **Memory Complexity:** $O(1)$. No additional memory is required beyond temporary variables.
*   **Strengths:** Numerically stable for non-degenerate triangles due to the use of coordinate differences. Simple to implement.
*   **Limitations:** Unstable for degenerate (collinear) triangles. The circum-center can be arbitrarily far for obtuse triangles, necessitating a fallback strategy.
*   **When to use:** When defining bounding spheres for triangle primitives in volume rendering (e.g., in TetSplat).
*   **When not to use:** For degenerate triangles or when a tighter bounding sphere is required for obtuse triangles without the fallback approximation.

## Background
The method is described in the context of "TetSplat," a technique for real-time rendering and volume clipping of large triangle meshes using splatting. The approach uses circum-centers of triangles (and tetrahedra) to define leaf nodes in a hierarchical data structure. The formula for the triangle circum-center is cited from reference [19] in the source paper, which is likely a standard computational geometry reference, but the specific implementation details and the obtuse fallback are attributed to Museth and Lombeyda [S1, unofficial].

## Sources
[S1] TetSplat: Real-time Rendering and Volume Clipping of Large ... - Museth, https://ken.museth.org/Publications_files/Museth-Lombeyda_VIS04.pdf, fetched 2026-10-28 (unofficial)
