# Extended Trilinear Triangulation

Also called: Extended MC33, Extended Marching Cubes 33, Extended triangulation to the Marching Cubes 33 algorithm

## Overview
Extended Trilinear Triangulation is a method for generating topologically correct, degenerate-free isosurface triangulations from volumetric data by extending the Marching Cubes 33 (MC33) algorithm [S1]. It solves the dual problems of topological ambiguity in the original Marching Cubes and the generation of degenerate or "skinny" triangles when vertex values exactly match the isovalue [S1]. Unlike standard MC33, which uses a fixed lookup table of 33 entries based on binary vertex signs, this method introduces a ternary classification ('+', '-', '=') and constructs triangulations using convex hulls of intersection points and vertex groups, ensuring the mesh topology coincides exactly with the trilinear interpolant [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A rectilinear grid $H$ with a scalar field $f$, an isovalue $\alpha$, and a unit cube $C$ with 8 vertices $v_0, \dots, v_7$ [S1].
*   **Output:** A set of triangles approximating the isosurface $\bar{S}_\alpha = \{x \in \mathbb{R}^3 | g(x) = \alpha\}$ within the cube, where $g$ is the trilinear interpolant [S1].

**Data Structures:**
*   **Vertex Label:** Each vertex $v_i$ is assigned a label $L(v_i) \in \{+, -, =\}$ [S1].
*   **Group of Vertices (GV):** A set of vertices in a cube connected by a "positive path" (for positive cubes) or "negative path" (for negative cubes) restricted to the cube's edges and faces [S1].
*   **Group of Edges (GE):** A set of edges intersected by the isosurface, incident to vertices in the same GV [S1].
*   **Intersection Points:** Midpoints of intersected edges (used for convex hull construction) [S1].

**Formulas:**
The trilinear interpolant $g(x,y,z)$ for a unit cube is:
$$
\begin{aligned}
g(x,y,z) &= (1-x)(1-y)(1-z)f(v_0) + (1-x)(1-y)zf(v_4) \\
&\quad + (1-x)y(1-z)f(v_3) + (1-x)yzf(v_7) \\
&\quad + x(1-y)(1-z)f(v_1) + x(1-y)zf(v_5) \\
&\quad + xy(1-z)f(v_2) + xyzf(v_6)
\end{aligned}
$$
where $v_0$ is at $(0,0,0)$, $v_1$ at $(1,0,0)$, $v_2$ at $(1,1,0)$, $v_3$ at $(0,1,0)$, $v_4$ at $(0,0,1)$, $v_5$ at $(1,0,1)$, $v_6$ at $(1,1,1)$, and $v_7$ at $(0,1,1)$ [S1].

**Steps:**

1.  **Classify Vertices:** For each vertex $v_i$ of the cube:
    *   If $f(v_i) > \alpha$, label $L(v_i) = '+'$ [S1].
    *   If $f(v_i) < \alpha$, label $L(v_i) = '-'$ [S1].
    *   If $f(v_i) = \alpha$, label $L(v_i) = '='$ [S1].

2.  **Classify Cube Sign:**
    *   Count $N_{=+}$ as the number of vertices labeled '=' or '+' [S1].
    *   If $N_{=+} \le 4$, the cube is **Positive** [S1].
    *   Otherwise, the cube is **Negative** [S1].

3.  **Identify Groups:**
    *   **Positive Cube:** Identify **Groups of Vertices (GV)** as sets of '+' vertices connected by a path of '+' vertices along the cube's edges and faces [S1]. Identify **Groups of Edges (GE)** as sets of intersected edges incident to vertices in the same GV [S1].
    *   **Negative Cube:** Apply the analogous definition using '-' vertices and negative paths [S1].

4.  **Interior Test:** Determine if groups of vertices are separated or joined inside the cube using the Chernyaev/Custodio interior test [S1]. Classify the cube into one of three categories: **Simple Leaf**, **Tunnel**, or **Interior Point** [S1]. Note that cases 6.1.1 and 6.1.2 have identical vertex/edge groups but different topologies; the interior test distinguishes them [S1].

5.  **Generate Triangulation:** Based on the category and specific case (e.g., Case 13 subcases), apply one of the following methods:

    *   **Simple Leaves Triangulation:**
        For each group $i$:
        1.  Let $G_i = GV_i \cup GE_i$ [S1].
        2.  Let $W_i$ be the set of intersection points (midpoints of edges in $G_i$) and vertices in $GV_i$ labeled '=' or '+' [S1].
        3.  Construct $CH_i = \text{conv}(W_i)$ [S1].
        4.  If $CH_i$ is 3D, triangulate the boundary $\partial(CH_i)$ and remove triangles lying on the cube faces [S1].
        5.  If $CH_i$ is 2D (contained in faces), return the triangulation of $CH_i$ [S1].
        6.  **Exception:** If all cube vertices are '=', no triangulation is generated to avoid duplicates [S1].
        7.  **Global Check:** Track grid faces to ensure a face is not triangulated twice by adjacent cubes [S1].

    *   **Tunnel Triangulation:**
        1.  Let $W$ be the set of midpoints of *all* intersected edges in the cube [S1].
        2.  Construct $CH = \text{conv}(W)$ [S1].
        3.  If $CH$ is 3D, triangulate $\partial(CH)$ [S1].
        4.  Remove triangles where all three vertices belong to the same Group of Edges $GE_i$ [S1].

    *   **Interior Point Leaves Triangulation:**
        1.  Identify faces where all four edges are intersected [S1].
        2.  Connect pairs of intersected edges on these faces based on bilinear interpolant behavior to form a simple closed polygonal curve [S1].
        3.  Add a vertex at the cube center [S1].
        4.  Form triangles connecting the center vertex to the segments of the polygonal curve [S1].

    *   **Case 13 Subcases (Complex):**
        *   **13.3:** One group of 4 positive vertices. Use the incident negative vertex. Apply Interior Point Leaves triangulation combined with Simple Leaves triangulation on the single negative vertex group [S1].
        *   **13.5.1:** Two groups of vertices. Apply Simple Leaves triangulation to both groups [S1].
        *   **13.5.2:** Two groups connected by interior, with signal change diagonally opposite to one vertex group. Apply Tunnel triangulation combined with Simple Leaves triangulation in the region diagonally opposite to the one vertex positive group [S1].

**Pitfalls and Edge Cases:**
*   **Degenerate Triangles:** Occur in standard MC if $f(v_i) = \alpha$ is treated as positive, creating a triangle with three vertices at the same point. The '=' label prevents this [S1].
*   **Skinny Triangles:** Occur when two vertices are close to a grid vertex and the third is far. The convex hull method mitigates this by respecting the interpolant's regions [S1].
*   **Duplicate Faces:** If a cube has all vertices '=', no mesh is generated. The algorithm tracks grid faces already triangulated to prevent a cube face from being triangulated twice by adjacent cubes [S1].

## Subtopics

### [Marching Cubes 33](2129-marching-cubes-33.md)
Resolves topological ambiguities of original MC using 33 entries derived from analytical tests. Unlike Extended Trilinear Triangulation, it does not use convex hulls or the '=' label, relying instead on a larger lookup table.

## Operating characteristics
*   **Complexity:** The method processes each cube independently [S1]. The complexity is dominated by the interior test and convex hull construction, which are constant time per cube for a fixed number of vertices (8) [S1].
*   **Performance:** In a 5x5x5 test case, the method produced correct Betti numbers (b0=1, b1=0, b2=1), whereas Raman and Wenger's method (which covers only 23 of 33 behaviors) failed, producing (b0=4, b1=0, b2=4) [S1].
*   **Strengths:** Guarantees topological correctness for all 33 cases of the trilinear interpolant; eliminates degenerate triangles; improves mesh quality by avoiding skinny triangles [S1].
*   **Limitations:** More complex implementation than standard MC due to group identification and interior tests; requires tracking of grid faces to prevent duplicates [S1].

## Background
The base Marching Cubes algorithm was proposed by Lorensen and Cline in 1987 [S1]. Marching Cubes 33 was proposed by Chernyaev in 1995 to guarantee topological correctness [S1]. Raman and Wenger introduced the extended lookup table concept with '+'/'-'/'=' labels in 2008 to avoid degenerate triangles, but their table only covered 23 of 33 behaviors, leading to topological errors in ambiguous cases (e.g., Case 10) [S1]. Extended Trilinear Triangulation extends MC33 by integrating the '=' label with the full topological correctness of MC33 using convex hulls [S1].

## Sources
[S1] An extended triangulation to the Marching Cubes 33 algorithm, https://doi.org/10.1186/s13173-019-0086-6, fetched 2026-10-26
