# Barycentric Thick Endpoint Calculation

Also called: Barycentric Endpoint Calculation

## Overview
Calculates the near ($f$) and far ($b$) endpoints of the "thick line" (the intersection of the line of sight with the tetrahedron volume) using the barycentric coordinates of the x-ray source. This method avoids projecting vertices to the image plane, allowing the calculation to be performed independently of the view transform. It builds on [Point-in-Tetrahedron Test via Barycentric Coordinates](2591-point-in-tetrahedron-test-via-barycentric-coordinates.md) by utilizing the source's barycentric vector $\mathbf{t}$ to determine the entry and exit points of the ray through the tetrahedron.

## How it works
**Inputs:**
*   Tetrahedron vertices in Cartesian coordinates: $v_0, v_1, v_2, v_3$.
*   Barycentric coordinates of the x-ray source $s$ with respect to the tetrahedron: $\mathbf{t} = (t_0, t_1, t_2, t_3)^T$.

**Outputs:**
*   Near endpoint position: $f$.
*   Far endpoint position: $b$.
*   (Optional) Barycentric coordinates of endpoints: $\mathbf{u}_f, \mathbf{u}_b$.

**Data Structures:**
*   $\mathbf{t}$: A 4-component vector representing the source's position in the tetrahedron's local barycentric frame.
*   $v_i$: 3D Cartesian vectors.

**Steps:**
1.  **Compute Near Endpoint ($f$):**
    The near endpoint lies on the intersection of the edge $(v_0, v_2)$ and the plane through $(s, v_1, v_3)$. Its Cartesian position is calculated as:
    $$ f = \frac{t_0}{t_0 + t_2} v_0 + \frac{t_2}{t_0 + t_2} v_2 $$
    Alternatively, using the opposite edge:
    $$ f = \frac{-t_1}{t_0 + t_2} v_1 + \frac{-t_3}{t_0 + t_2} v_3 $$
    The barycentric coordinates of $f$ are:
    $$ \mathbf{u}_f = (\alpha_0, \alpha_1, \alpha_2, \alpha_3) = \left( \frac{t_0}{t_0 + t_2}, \frac{-t_1}{t_0 + t_2}, \frac{t_2}{t_0 + t_2}, \frac{-t_3}{t_0 + t_2} \right) $$
    Note: $\alpha_2$ is derived from the constraint $\sum \alpha_i = 1$, but the formula above explicitly shows the weights for $v_0$ and $v_2$ in the first expression. The source text defines the tuple $(\alpha_0, \alpha_1, \alpha_3)$ as $(t_0 + t_2)^{-1}(t_0, -t_1, -t_3)$, implying $\alpha_2$ is the remaining weight to sum to 1, or specifically that the position is a convex combination of $v_0, v_2$ with weights proportional to $t_0, t_2$.

2.  **Compute Far Endpoint ($b$):**
    The far endpoint lies on the intersection of the edge $(v_1, v_3)$ and the plane through $(s, v_0, v_2)$. Its Cartesian position is:
    $$ b = \frac{t_1}{t_1 + t_3} v_1 + \frac{t_3}{t_1 + t_3} v_3 $$
    The relationship between $f$ and $b$ is:
    $$ b = -\frac{t_0 + t_2}{t_1 + t_3} f $$
    This indicates that $f$ and $b$ are scaled versions of each other and project to the same image point.

3.  **Restore Winding Order:**
    After determining the rendering order, swap the index-0 component and the index-$j_0$ component in the "barycentric coordinates" attributes of each vertex to restore them to the original winding order [S1].

**Formulas Summary:**
*   $f = \frac{t_0}{t_0 + t_2} v_0 + \frac{t_2}{t_0 + t_2} v_2$
*   $b = \frac{t_1}{t_1 + t_3} v_1 + \frac{t_3}{t_1 + t_3} v_3$
*   $\mathbf{u}_f = (t_0 + t_2)^{-1}(t_0, -t_1, t_2, -t_3)$ (Note: The source text lists $(\alpha_0, \alpha_1, \alpha_3)$, but the position formula uses $v_0, v_2$ with weights $t_0, t_2$. The barycentric vector must sum to 1. If $\alpha_0 = t_0/(t_0+t_2)$ and $\alpha_2 = t_2/(t_0+t_2)$, then $\alpha_1 + \alpha_3 = 0$. The source gives $\alpha_1 = -t_1/(t_0+t_2)$ and $\alpha_3 = -t_3/(t_0+t_2)$. Sum: $(t_0 - t_1 + t_2 - t_3)/(t_0+t_2)$. Since $\sum t_i = 1$ is not necessarily true for source barycentrics in this context (they are homogeneous or derived from inverse matrix), the explicit position formulas are the primary implementation targets.)

**Pitfalls:**
*   **Division by Zero:** The denominators $(t_0 + t_2)$ and $(t_1 + t_3)$ are guaranteed to be non-zero. This is because $t_0$ and $t_2$ have equal signs, and $t_1$ and $t_3$ have opposite signs [S1].

## Subtopics

### Orthographic Projection Thick Endpoint Calculation
Determines thick endpoints in orthographic projection by projecting the Z-axis direction through the inverse tetrahedron matrix $M_T^{-1}$ to get a difference vector $\mathbf{d}$ in barycentric space.
*   **Definition:** The difference vector $\mathbf{d}$ represents the change in barycentric weights as we move along the Z direction. It is calculated as $\mathbf{d} = M_T^{-1}(0, 0, 1, 0)^T$. The actual difference vector in barycentric space for the endpoints is $\mathbf{d}' = \alpha \mathbf{d}$.
*   **Steps:**
    1.  Compute $\mathbf{d} = M_T^{-1}(0, 0, 1, 0)^T$.
    2.  Analyze the signs of the components of $\mathbf{d} = (d_0, d_1, d_2, d_3)^T$.
    3.  **Triangle Case:** If one sign is strictly positive (or negative) and the others are nonpositive (nonnegative), the corresponding vertex is a thick endpoint.
        *   If vertex 0 is the near endpoint ($\mathbf{u}_0 = (1, 0, 0, 0)^T$), the far endpoint barycentric coordinates are $\mathbf{u}_b = d_0^{-1}(0, -d_1, -d_2, -d_3)^T$. This is derived from $\alpha = -d_0^{-1}$.
    4.  **Quadrilateral Case:** If two signs are positive and two are negative, the silhouette is a quadrilateral.
        *   Near endpoint barycentric coordinates: $\mathbf{u}_f = (d_0 + d_2)^{-1}(d_0, 0, d_2, 0)^T$.
        *   Far endpoint barycentric coordinates: $\mathbf{u}_b = (d_1 + d_3)^{-1}(0, d_1, 0, d_3)^T$.
*   **When to choose:** Use this variant when the projection is orthographic (line of sight parallel to Z-axis) rather than perspective. It relies on the tetrahedron matrix $M_T$ and its inverse.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per tetrahedron, involving a constant number of arithmetic operations.
*   **Memory Complexity:** $O(1)$ auxiliary space.
*   **Strengths:** Independent of the view transform; can be computed using only the tetrahedron's local barycentric coordinates of the source. Avoids expensive projection operations for endpoint calculation.
*   **Limitations:** Requires the barycentric coordinates of the source $\mathbf{t}$ (or the matrix $M_T$ for orthographic case). Precision issues may arise if $t_0 + t_2$ or $t_1 + t_3$ are close to zero, though the sign properties guarantee they are not zero.
*   **When to use:** When efficient ray-tetrahedron intersection is needed for volume rendering, particularly in projected tetrahedra algorithms.

## Background
Originated in the "Projected Tetrahedra Revisited" paper by R. H. et al. (2006) [S1]. The method leverages the property that the barycentric coordinates of the x-ray source with respect to the tetrahedron are independent of the view transform, allowing for efficient calculation of the thick line endpoints (the intersection segment of the ray with the tetrahedron). It is used in volume rendering techniques that rely on projected tetrahedra.

## Sources
[S1] Projected Tetrahedra Revisited: A Barycentric Formulation Applied to ..., https://www.cs.jhu.edu/~rht//RHT%20Papers/2006/Projected%20Tetrahedra%20Revisited.pdf, fetched 2026-10-27
