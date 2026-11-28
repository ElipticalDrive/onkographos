# Barycentric Projected Tetrahedron Tessellation

Also called: Barycentric PT, PT in Perspective Projection

## Overview
Barycentric Projected Tetrahedron Tessellation is a volume rendering method for unstructured tetrahedral grids that determines the silhouette shape and tessellation of a projected tetrahedron under perspective projection by computing the barycentric coordinates of the eye position with respect to the tetrahedron vertices [S1]. It differs from the classic Projected Tetrahedra (PT) method by using barycentric coordinates to ensure geometrically correct interpolation of thickness and density under perspective projection, computing integrals on the GPU fragment unit using a closed formula rather than lookup tables [S1]. Unlike Shirley and Tuchman or Wylie, this method works under perspective projection without requiring perspective division before line intersection tests or back-projection [S1].

## How it works
**Inputs:**
- Four tetrahedron vertices $v_0, v_1, v_2, v_3$ in homogeneous coordinates (eye-space).
- Eye position $s = (0, 0, 0, 1)^T$ in homogeneous coordinates.

**Outputs:**
- A tessellation of the projected tetrahedron defined by thick endpoints $f$ and $b$ in Cartesian coordinates, and the rendering order of vertices.

**Data Structures:**
- Vertex Matrix $M_T$: A $4 \times 4$ matrix where each column is a vertex in homogeneous coordinates.
  $$ M_T = [v_0 \ v_1 \ v_2 \ v_3] $$
- Barycentric Coordinate Vector $T$: A $4 \times 1$ vector $T = (t_0, t_1, t_2, t_3)^T$.

**Steps:**

1. **Compute Barycentric Coordinates:**
   Calculate the barycentric coordinates $T$ of the eye position $s$ with respect to the tetrahedron vertices by solving the linear system:
   $$ T = M_T^{-1} s $$
   This yields $T = (t_0, t_1, t_2, t_3)^T$.

2. **Determine Silhouette Case:**
   Analyze the signs of the components $t_j$ of $T$ to determine the geometric relationship between the eye and the tetrahedron. Ignore degenerate cases (where any $t_j = 0$) as they are assumed to be automatically culled by the rendering pipeline [S1].
   - **Interior:** If all $t_j \ge 0$, the eye is inside or on the boundary of the tetrahedron. Do not render.
   - **Vertex Case:** If exactly one $t_{j_0} > 0$ and the others are $\le 0$, the eye is in front of vertex $v_{j_0}$. The silhouette is a triangle.
   - **Face Case:** If exactly one $t_{j_0} < 0$ and the others are $\ge 0$, the eye is in front of face $F_{j_0}$. The silhouette is a triangle.
   - **Quadrilateral Case:** If $t_{j_0}$ and $t_{j_2}$ have one sign, while $t_{j_1}$ and $t_{j_3}$ have the opposite sign, the silhouette is a quadrilateral. The eye is in front of both edges $(v_{j_0}, v_{j_2})$ and $(v_{j_1}, v_{j_3})$.

3. **Reorder Vertices:**
   Reorder the vertices into a canonical rendering order to simplify endpoint calculation.
   - **Triangle Case:** Swap $v_0$ and $v_{j_0}$ so that $v_0$ corresponds to the vertex on the thick line (the "near" vertex).
   - **Quadrilateral Case:** Swap $v_0$ with $v_{j_0}$ (where $j_0 \neq 0, 2$) such that $\text{sign}(t_0) \neq \text{sign}(t_2)$ and $\text{sign}(t_2) = \text{sign}(t_{j_0})$. This places $v_0$ and $v_2$ at opposite corners of the quadrilateral silhouette.

4. **Compute Thick Endpoints:**
   Calculate the near ($f$) and far ($b$) endpoints of the thick line in Cartesian coordinates.
   - **Triangle Case:**
     Set the barycentric coordinates of the thick endpoint $v_0$ to $u_0 = (1, 0, 0, 0)^T$.
     Compute the parameter $\beta$ for the other endpoint $b$ on face $F_0$:
     $$ \beta = \frac{-t_0}{1 - t_0} $$
     The endpoint $b$ is then derived from the face geometry (specifically, the intersection of the ray from the eye through the projected vertex with the opposite face).
   - **Quadrilateral Case:**
     Compute the first thick endpoint $f$ (intersection of edge $(v_0, v_2)$ and plane through $s, v_1, v_3$):
     $$ f = \frac{t_0}{t_0 + t_2} v_0 + \frac{t_2}{t_0 + t_2} v_2 $$
     Verify $f$ also lies on the plane through $s, v_1, v_3$:
     $$ f = \frac{-t_1}{t_0 + t_2} v_1 + \frac{-t_3}{t_0 + t_2} v_3 $$
     Compute the second thick endpoint $b$ (intersection of edge $(v_1, v_3)$ and plane through $s, v_0, v_2$):
     $$ b = \frac{t_1}{t_1 + t_3} v_1 + \frac{t_3}{t_1 + t_3} v_3 $$
     Verify $b$ is collinear with $f$ and the eye:
     $$ b = -\frac{t_0 + t_2}{t_1 + t_3} f $$

**Pitfalls:**
- **Degeneracy:** Cases where some $t_j$ are zero correspond to 1D or 0D regions. These are ignored and assumed to be automatically culled in the rendering pipeline [S1].
- **Perspective Division:** Unlike previous methods, this method avoids the need for perspective division before line intersection tests or back-projection, reducing computational complexity and potential precision errors [S1].

## Subtopics

### [Projected Tetrahedra (PT) with Barycentric Formulation](23668-projected-tetrahedra-pt-with-barycentric-formulation.md)
A variant that computes line integrals of higher-order attenuation functions (specifically Bernstein polynomials in barycentric coordinates) without preintegration or depth sorting. It differs from the classic PT method by using barycentric coordinates to ensure geometrically correct interpolation of thickness and density under perspective projection, and it computes integrals on the GPU fragment unit using a closed formula rather than lookup tables.

## Operating characteristics
- **Time Complexity:** Determining the silhouette case requires computing $T = M_T^{-1} s$, which involves a $4 \times 4$ matrix inversion and a vector-matrix multiplication. This is comparable in operation count to Shirley and Tuchman's method of checking relative directions of face normals and view direction [S1].
- **Memory Complexity:** Minimal additional memory is required beyond the vertex attributes. The inclusion of barycentric coordinates in vertex attributes could potentially increase algorithm complexity compared to "standard" geometrical models, but the derivation in barycentric coordinates contributes to efficiency [S1].
- **Strengths:** Works directly under perspective projection without requiring perspective division before line intersection tests or back-projection [S1]. Provides direct computation of near and far endpoints of the thick line without explicit line intersection calculations [S1].
- **Limitations:** Degenerate cases must be handled by the rendering pipeline (culling) [S1].

## Background
This method generalizes observations from Shirley and Tuchman's PT method and Wylie's tessellation algorithm to perspective projection [S1]. Previous attempts to implement line-intersection tests similar to Wylie's for the perspective case proved difficult, requiring perspective division before finding the projection of the thick point, and then back-projection and computation of line intersections in space to find the true coordinates and thickness at the thick vertex [S1]. The solution proposed here is simpler and uses barycentric coordinates to solve the problem under perspective projection, with a slight variation under orthographic projection [S1].

## Sources
[S1] Projected Tetrahedra Revisited: A Barycentric Formulation Applied to ..., https://www.cs.jhu.edu/~rht//RHT%20Papers/2006/Projected%20Tetrahedra%20Revisited.pdf, fetched 2026-10-27
