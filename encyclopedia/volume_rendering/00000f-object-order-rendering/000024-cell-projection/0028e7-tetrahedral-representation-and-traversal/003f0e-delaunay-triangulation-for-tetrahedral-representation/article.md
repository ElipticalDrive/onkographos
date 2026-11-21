# Delaunay Triangulation for Tetrahedral Representation

Also called: 3D Delaunay Triangulation, gDel3D-based Tetrahedralization

## Overview
3D Delaunay Triangulation constructs a surface by defining tetrahedra from arbitrarily distributed 3D points, characterized by the property that no point lies within the circumsphere of any tetrahedron [S2]. It solves the problem of surface reconstruction from volumetric data or point clouds by enforcing geometric optimality (empty circumsphere property) [S2]. Unlike [Projected Tetrahedra](877-projected-tetrahedra.md) which projects pre-defined grid cells, or [Marching Tetrahedrons Algorithm](940-marching-tetrahedrons-algorithm.md) which operates on a fixed lattice, this method adapts the tetrahedral mesh to the distribution of input sites [S2]. It is distinguished from its sibling [Local Convex Hull (LCH)](11862-local-convex-hull-lch.md) by using Delaunay triangulation rather than a convex hull algorithm for local surface construction, offering a trade-off between processing time and shape detail preservation [S1].

## How it works
**Inputs:** A set of 3D points (Voronoi sites) $P = \{p_1, p_2, ..., p_n\}$ representing the volumetric data or surface samples.
**Outputs:** A tetrahedral mesh $T$ where each tetrahedron $t \in T$ satisfies the empty circumsphere property.

**Data Structures:**
*   **Points:** Array of 3D coordinates $(x, y, z)$.
*   **Tetrahedra:** List of indices referencing the points array. Each tetrahedron is defined by 4 vertex indices $(v_1, v_2, v_3, v_4)$.
*   **Circumsphere:** For each tetrahedron, the center $(c_x, c_y, c_z)$ and squared radius $r^2$ of the unique sphere passing through its four vertices.

**Algorithm Steps:**
1.  **Initialization:** Create a "super tetrahedron" that encloses all input points $P$. This ensures the triangulation starts with a valid boundary [S2].
2.  **Triangulation:** Construct the initial Delaunay triangulation of the points. This can be done using standard incremental insertion or divide-and-conquer algorithms. In GPU-accelerated implementations, libraries such as gDel3D are used to accelerate this computation [S1].
3.  **Empty Circumsphere Check:** For each tetrahedron $t$ in the mesh, calculate its circumsphere. Check if any point $p \in P$ lies strictly inside this circumsphere.
    *   The condition for a point $p$ being inside the circumsphere of tetrahedron $t$ with vertices $v_1, v_2, v_3, v_4$ is determined by the determinant:
        $$
        \det \begin{pmatrix}
        x_p & y_p & z_p & x_p^2 + y_p^2 + z_p^2 & 1 \\
        x_1 & y_1 & z_1 & x_1^2 + y_1^2 + z_1^2 & 1 \\
        x_2 & y_2 & z_2 & x_2^2 + y_2^2 + z_2^2 & 1 \\
        x_3 & y_3 & z_3 & x_3^2 + y_3^2 + z_3^2 & 1 \\
        x_4 & y_4 & z_4 & x_4^2 + y_4^2 + z_4^2 & 1
        \end{pmatrix} > 0
        $$
        (Assuming a right-handed coordinate system and counter-clockwise vertex ordering) [S2].
4.  **Subdivision:** If a point $p$ is found within the circumsphere of tetrahedron $t$, subdivide $t$. This involves removing $t$ and creating new tetrahedra connecting $p$ to the faces of $t$ that are visible from $p$, or using a flip-based approach to restore the Delaunay property [S2].
5.  **Iteration:** Repeat steps 3 and 4 until no tetrahedron contains a point in its circumsphere. In differentiable rendering contexts (e.g., DCCVT), the triangulation is recomputed from scratch at every iteration to allow gradient flow [S1].

**Parameters:**
*   **Super Tetrahedron Size:** Must be large enough to enclose all input points. Exact size is not given in the sources [S2].
*   **Iteration Count:** In DCCVT experiments, a fixed 1k iterations are used [S1].

**Edge Cases and Pitfalls:**
*   **Constrained Delaunay Triangulation (CDT):** Standard DT cannot tetrahedralize some special polyhedra (e.g., twisted prisms) without additional points or surface modification. The problem of determining whether a given polyhedron can be tetrahedralized is NP-complete [S2].
*   **Precision:** Floating-point errors in determinant calculations can lead to incorrect circumsphere tests. Robust predicates are recommended [G].
*   **Degenerate Input:** Collinear or coplanar points can cause degenerate tetrahedra. Pre-processing to remove duplicates or handle degeneracies is necessary [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def delaunay_tetrahedralization(points):
    """
    Constructs a 3D Delaunay triangulation from a set of 3D points.
    """
    # Step 1: Initialize with a super tetrahedron
    super_tetra = create_super_tetrahedron(points)
    tetrahedra = [super_tetra]
    
    # Step 2: Insert points incrementally
    for p in points:
        # Find all tetrahedra whose circumsphere contains p
        affected_tetras = []
        for t in tetrahedra:
            if is_point_in_circumsphere(p, t):
                affected_tetras.append(t)
        
        # Remove affected tetrahedra and create new ones
        # This is a simplified representation; actual implementation 
        # requires handling the boundary of the hole created by removal
        tetrahedra = remove_tetrahedra(tetrahedra, affected_tetras)
        new_tetras = create_new_tetrahedra(p, affected_tetras)
        tetrahedra.extend(new_tetras)
        
    # Remove the super tetrahedron if it remains
    tetrahedra = remove_super_tetrahedron(tetrahedra)
    return tetrahedra

def is_point_in_circumsphere(p, t):
    """
    Checks if point p is inside the circumsphere of tetrahedron t.
    """
    v1, v2, v3, v4 = t.vertices
    det = determinant_matrix(p, v1, v2, v3, v4)
    return det > 0

def determinant_matrix(p, v1, v2, v3, v4):
    """
    Computes the determinant for the circumsphere test.
    """
    # Construct the 5x5 matrix
    M = [
        [p.x, p.y, p.z, p.x**2 + p.y**2 + p.z**2, 1],
        [v1.x, v1.y, v1.z, v1.x**2 + v1.y**2 + v1.z**2, 1],
        [v2.x, v2.y, v2.z, v2.x**2 + v2.y**2 + v2.z**2, 1],
        [v3.x, v3.y, v3.z, v3.x**2 + v3.y**2 + v3.z**2, 1],
        [v4.x, v4.y, v4.z, v4.x**2 + v4.y**2 + v4.z**2, 1]
    ]
    return compute_determinant(M)
```

## Subtopics

### [Local Convex Hull (LCH)](11862-local-convex-hull-lch.md)
Local Convex Hull (LCH) is a surface reconstruction method that constructs triangular patches from volumetric data by estimating surfaces from local convexities, assuming the object is piecewise convex and surfaces are tightly connected. It solves the problem of discontinuous surfaces and holes common in simpler methods like Marching Cubes (MC) and Visual Hull Methods (VMC) by enforcing connectivity and continuity properties. LCH is distinguished from its sibling [Delaunay Triangulation for Tetrahedral Representation](9905-delaunay-triangulation-for-tetrahedral-representation.md) (specifically Local DT) by using a convex hull algorithm rather than Delaunay triangulation for local surface construction, offering a trade-off between processing time and shape detail preservation.

### [Parallel Voxel Carving for Multi-view Stereo](14743-parallel-voxel-carving-for-multi-view-stereo.md)
Parallel Voxel Carving is a scalable volumetric reconstruction algorithm for multi-view stereo that utilizes a GPU to simultaneously execute a large number of threads, each performing voxel carving to integrate depth maps with images from multiple views. It solves the problem of reconstructing large-scale outdoor scenes by converting semi-dense image correspondences into view-dependent depth maps and then projecting these maps into a 3D voxel grid to carve away photo-inconsistent volumes. Unlike direct mesh integration, which struggles with triangle intersections in parallel environments, this method outputs surface voxels with normals and connectivity, making it suitable for point-based rendering and scalable to high-resolution grids via streaming.

## Operating characteristics
*   **Time Complexity:** The worst-case time complexity for 3D Delaunay triangulation is $O(n^2)$, but average-case is $O(n \log n)$ for random points [S2]. In differentiable rendering contexts, recomputing from scratch at every iteration increases cost [S1].
*   **Memory Complexity:** $O(n)$ for the number of tetrahedra in 3D, proportional to the number of input points [S2].
*   **Measured Performance:** In DCCVT, gDel3D is used to accelerate computation on the GPU [S1]. Experiments are executed for a fixed 1k iterations [S1].
*   **Strengths:** Produces high-quality tetrahedral meshes with good geometric properties (empty circumsphere). Handles arbitrarily distributed points [S2].
*   **Limitations:** Cannot handle constrained boundaries easily without additional modifications (NP-complete for general polyhedra) [S2]. Computationally expensive for large datasets if not accelerated [S1].
*   **When to use:** When high-quality tetrahedralization of arbitrary point clouds is required, and constraints are not a primary concern.
*   **When not to use:** When constrained boundaries are essential and the geometry is complex (e.g., twisted prisms) [S2].

## Background
The 3D Delaunay Triangulation (DT) constructs a surface by defining tetrahedrons from arbitrarily distributed 3D points [S2]. The Constrained Delaunay Triangulation (CDT) has been evolved to include a predescribed boundary [S2]. Although other variations like conforming constrained DT have been proposed, they assume initial boundary information is given [S2]. The problem of determining whether a given polyhedron can be tetrahedralized is NP-complete [S2]. In recent applications, such as DCCVT, DT is used to obtain a tetrahedral representation of Voronoi sites, accelerated by gDel3D on the GPU [S1].

## Sources
[S1] DCCVT: Differentiable Clipped Centroidal Voronoi Tessellation, https://arxiv.org/abs/2601.13603v1, fetched 2026-10-26
[S2] Local hull-based surface construction of volumetric data from silhouettes, https://doi.org/10.1109/tip.2008.926149, fetched 2026-10-30
