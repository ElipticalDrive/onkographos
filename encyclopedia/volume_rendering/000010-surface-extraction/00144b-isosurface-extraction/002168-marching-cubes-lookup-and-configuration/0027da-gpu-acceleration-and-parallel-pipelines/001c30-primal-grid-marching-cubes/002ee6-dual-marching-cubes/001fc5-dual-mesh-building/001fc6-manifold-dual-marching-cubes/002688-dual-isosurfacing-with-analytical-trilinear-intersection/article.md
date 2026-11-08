# Dual Isosurfacing with Analytical Trilinear Intersection

Also called: Analytical Isosurface Extraction, Ray Intersection Isosurfacing, Topologically Accurate Dual Isosurfacing, Trilinear Ray Intersection

## Overview
Dual Isosurfacing with Analytical Trilinear Intersection extracts the topology of an analytical isosurface defined by a trilinear function within a voxel cell by determining the optimal set of diagonals to intersect with rays, ensuring one point per disjoint surface component [S1][S2]. Unlike standard Marching Cubes which uses linear interpolation and fixed lookup tables, this method uses the trilinear function exhaustively to determine the topology of the analytical isosurface, allowing for more accurate diagonal selection [S1][S2]. It builds on [Manifold Dual Marching Cubes](2135-manifold-dual-marching-cubes.md) by resolving ambiguous configurations through analytical ray intersection rather than precomputed lookup tables [S1].

## How it works
### Inputs and Outputs
**Inputs:**
- Scalar field values at the 8 vertices of a voxel cell: $v_0, \dots, v_7$ [S1][S2].
- The isovalue $I$ [S1][S2].
- The cell sign configuration index $C \in [0, 13]$ derived from the signs of the 8 vertices relative to $I$ [S1][S2].

**Outputs:**
- A set of intersection points on the cell diagonals [S1][S2].
- For each point, the diagonal index and the parameter $t \in [0, 1]$ along the diagonal where the trilinear function equals $I$ [S1][S2].

### Data Structures
The method relies on the classification of the 14 sign-configurations (numbered 0 to 13) of the MC-case table into five categories based on the number of ambiguous faces: 0, 1, 2, 3, or 6 [S1][S2].

**Ambiguous Face (AF):** A cell face with four same-sign ended diagonals and four $(+/-)$ edges [S2]. An AF is separated with respect to a specific sign if the contour on the face is a rectangular hyperbola [S2]. The sign is determined by the vertices in the quadrants containing the hyperbola, using the asymptotic decider [S2].
- $(+)$ ambiguous-face: Separated with respect to positive sign [S2].
- $(-)$ ambiguous-face: Separated with respect to negative sign [S2].

**Topological Configuration Index:** Configurations are represented as $x$, $x.y$, or $x.y.z$, where $x$ is the sign configuration, $y$ is the topological configuration, and $z$ is the sub-configuration [S1][S2].

**DeVella's Necklace Test:** Used to distinguish topological sub-configurations (specifically for tunnels) by checking if two vertices on a diagonal are internally connected [S2]. A positive result indicates the presence of a tunnel [S1][S2].

### Numbered Steps

1.  **Determine Cell Sign and Ambiguous Faces:**
    - Calculate the sign of the cell: Positive if more vertices are positive, Negative if more are negative, Neutral if equal [S2].
    - Identify ambiguous faces on the cell boundaries using the asymptotic decider [S2].
    - Categorize the cell into one of the five groups based on the count of ambiguous faces (0, 1, 2, 3, or 6) [S1][S2].

2.  **Select Diagonals for Ray Intersection:**
    Based on the category and specific topological sub-configuration, select diagonals to intersect [S1]. The goal is one point per disjoint component, relaxed for tunnels to capture manifold complexity [S1][S2].

    **Category 0 Ambiguous Faces (Sign Configurations 0, 1, 2, 4, 5, 8, 9, 11):**
    - Ignore Case 0 (no intersection) [S1][S2].
    - For Cases 1, 2, 5, 8, 9, 11: Use one $(+/-)$ diagonal for intersection [S1][S2].
    - For Case 4:
        - Use DeVella's necklace test to distinguish sub-configurations 4.1 (disjoint surfaces) and 4.2 (tunnel) [S1][S2].
        - If 4.1: Use the $(-/-)$ diagonal [S1][S2].
        - If 4.2: Use 2 of the 3 $(+/+)$ diagonals to obtain multiple points for the tunnel [S1][S2].

    **Category 1 Ambiguous Face (Sign Configurations 3, 6):**
    - If a $(+)$ cell has a $(+)$ ambiguous face: Single isosurface component occurs [S2]. Use one $(+/-)$ diagonal [S1][S2].
    - If a $(+)$ cell has a $(-)$ ambiguous face: Two isosurface components occur [S1][S2].
        - Cases 3.1 and 6.1.1 (two disjoint components, no tunnels):
            - Use one $(-/-)$ diagonal to obtain two intersection points [S1][S2].
            - If no $(-/-)$ diagonal exists (Case 3.1): Use two $(+/-)$ diagonals to obtain one intersection point each [S1][S2].
        - Case 6.1.2 (tunnel): Use both $(+/+)$ diagonals to obtain four intersection points on the tunnel [S1][S2].
        - Cases 3.2 and 6.2 (single isosurface component): Use one $(+/-)$ diagonal [S1][S2].

    **Category 2 Ambiguous Faces (Sign Configurations 10, 12):**
    - If ambiguous faces have different signs: Single component occurs (Cases 10.2, 12.2) [S1]. Use one $(+/-)$ diagonal [S1][S2].
    - If ambiguous faces have the same sign:
        - Disjoint surfaces with two $(+)$ AF: Use one $(+/+)$ diagonal [S1][S2].
        - Disjoint surfaces with two $(-)$ AF: Use one $(-/-)$ diagonal [S1][S2].
        - Tunnel with two $(+)$ AF (Case 10.1.2): Use two $(-/-)$ diagonals [S1][S2].
        - Tunnel with two $(-)$ AF (Case 12.1.2): Use one $(-/-)$ diagonal and two $(+/-)$ diagonals [S1][S2].

    **Category 3 Ambiguous Faces (Sign Configuration 7):**
    - Case 7.1 (three $(+)$ AF): Three disjoint surfaces [S1]. Use three $(+/-)$ diagonals [S1][S2].
    - Case 7.2 (two $(+)$ AF, one $(-)$ AF): Two disjoint surfaces [S1]. Use two $(+/-)$ diagonals through the $(+)$ vertices in one of the $(+)$ ambiguous faces [S1][S2].
    - Case 7.3 (one $(+)$ AF, two $(-)$ AF): One surface [S1]. Use any one $(+/-)$ diagonal [S1][S2].
    - Case 7.4 (three $(-)$ AF):
        - Use DeVella's necklace test to distinguish 7.4.1 (disjoint surfaces) and 7.4.2 (tunnel) [S1][S2].
        - If 7.4.1: Use one $(+/+)$ diagonal to obtain two intersection points [S1][S2].
        - If 7.4.2: Use all three $(+/-)$ diagonals to obtain three points on the tunnel [S1][S2].

    **Category 6 Ambiguous Faces (Sign Configuration 13):**
    - Reduce seven possibilities to four cases using mirror symmetry pairs: (1) and (7), (2) and (6), (3) and (5) [S1][S2].
    - Case 13.1 (six $(+)$ AF): Four disjoint surfaces [S1]. Use all four $(+/-)$ diagonals [S1][S2].
    - Case 13.2 (five $(+)$ AF, one $(-)$ AF): Three disjoint surfaces [S1]. Use three $(+/-)$ diagonals, excluding the one through one of the two $(+)$ vertices in the $(-)$ ambiguous face [S1][S2].
    - Case 13.3 (four $(+)$ AF, two $(-)$ AF): Two disjoint surfaces [S1]. Find the $(+)$ vertex common to all three $(+)$ ambiguous faces [S1]. Use the $(+/-)$ diagonal through this vertex as one ray, and any one of the remaining three diagonals as the second ray [S1][S2].
    - Case 13.4 (equal $(+)$ and $(-)$ AF, common vertex signs differ from faces): One isosurface piece [S1]. Use any one of the four $(+/-)$ diagonals [S1][S2].
    - Case 13.5 (equal $(+)$ and $(-)$ AF, common vertex signs same as faces):
        - Use DeVella's necklace test to distinguish 13.5.1 (three disjoint surfaces) and 13.5.2 (two disjoint surfaces, one is a tunnel) [S1][S2].
        - If 13.5.1: Use the $(+/-)$ diagonal through the common vertex of the three same-signed ambiguous faces to get three points [S1][S2].
        - If 13.5.2: Use the diagonal through the common vertex of the three same-signed ambiguous faces as a ray [S1]. Use the remaining three diagonals as rays to find points for the tunnel [S1]. Store diagonal differences for connectivity [S1][S2].

3.  **Compute Intersection Points:**
    For each selected diagonal, solve the trilinear equation $f(t) = I$ along the diagonal to find the intersection point [S1]. The trilinear function $f(x,y,z)$ is defined by the vertex values [G]. Along a diagonal, this reduces to a cubic equation in one parameter $t$ [S1].

4.  **Output Points:**
    Return the set of computed intersection points [G].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dual_isosurfacing_analytical_trilinear(vertex_values, isovalue):
    """
    Extracts isosurface points for a single voxel cell using analytical trilinear intersection.
    
    Args:
        vertex_values: List of 8 scalar values [v0, ..., v7] at cell vertices.
        isovalue: The scalar isovalue I.
        
    Returns:
        List of intersection points (t, diagonal_index) for the cell.
    """
    # 1. Determine Cell Sign and Ambiguous Faces
    signs = [1 if v > isovalue else -1 for v in vertex_values]
    num_pos = sum(1 for s in signs if s == 1)
    num_neg = sum(1 for s in signs if s == -1)
    cell_sign = 1 if num_pos > num_neg else (-1 if num_neg > num_pos else 0)
    
    # Map 8-bit sign pattern to canonical MC cases 0-13 [G]
    C = compute_mc_case_index(signs)
    
    # Identify Ambiguous Faces (AF) using asymptotic decider [G]
    ambiguous_faces = []
    for face_idx in range(6):
        if is_ambiguous_face(vertex_values, face_idx, isovalue):
            af_sign = get_af_sign(vertex_values, face_idx, isovalue)
            ambiguous_faces.append((face_idx, af_sign))
    num_af = len(ambiguous_faces)
    
    selected_diagonals = []
    
    if num_af == 0:
        if C == 0: return []
        elif C == 4:
            is_tunnel = devellas_necklace_test(vertex_values, C)
            if not is_tunnel: selected_diagonals.append(3) # (-/-)
            else: selected_diagonals.extend([0, 1]) # 2 of 3 (+/+)
        else: selected_diagonals.append(2) # (+/-)
    elif num_af == 1:
        af_sign = ambiguous_faces[0][1]
        if cell_sign == af_sign: selected_diagonals.append(2) # Single component
        else:
            is_tunnel = devellas_necklace_test(vertex_values, C)
            if not is_tunnel:
                if has_neg_neg_diagonal(C): selected_diagonals.append(3)
                else: selected_diagonals.extend([0, 2]) # Case 3.1
            else: selected_diagonals.extend([0, 1]) # 6.1.2 tunnel
    elif num_af == 2:
        af_signs = [af[1] for af in ambiguous_faces]
        if af_signs[0] != af_signs[1]: selected_diagonals.append(2) # Different signs
        else:
            if af_signs[0] == 1: # Two (+) AF
                is_tunnel = devellas_necklace_test(vertex_values, C)
                if not is_tunnel: selected_diagonals.append(0) # (+/+)
                else: selected_diagonals.extend([3, 3]) # Tunnel (10.1.2)
            else: # Two (-) AF
                is_tunnel = devellas_necklace_test(vertex_values, C)
                if not is_tunnel: selected_diagonals.append(3) # (-/-)
                else: selected_diagonals.extend([3, 2, 2]) # Tunnel (12.1.2)
    elif num_af == 3:
        pos_af_count = sum(1 for s in [af[1] for af in ambiguous_faces] if s == 1)
        if pos_af_count == 3: selected_diagonals.extend([0, 1, 2]) # 7.1
        elif pos_af_count == 2: selected_diagonals.extend([0, 2]) # 7.2
        elif pos_af_count == 1: selected_diagonals.append(2) # 7.3
        else: # 7.4 (three (-) AF)
            is_tunnel = devellas_necklace_test(vertex_values, C)
            if not is_tunnel: selected_diagonals.append(0) # 7.4.1
            else: selected_diagonals.extend([0, 1, 2]) # 7.4.2
    elif num_af == 6:
        pos_af_count = sum(1 for s in [af[1] for af in ambiguous_faces] if s == 1)
        if pos_af_count in (0, 6): selected_diagonals.extend([0, 1, 2, 3]) # 13.1 / 13.7
        elif pos_af_count in (1, 5): selected_diagonals.extend([0, 1, 2]) # 13.2 / 13.6
        elif pos_af_count in (2, 4): # 13.3 / 13.5
            common_vertex = find_common_vertex(vertex_values, [af[1] for af in ambiguous_faces])
            selected_diagonals.extend([2, 0]) # Through common & any remaining
        else: # pos_af_count == 3 (13.4 / 13.5)
            is_tunnel = devellas_necklace_test(vertex_values, C)
            if not is_tunnel:
                if pos_af_count == 3 and cell_sign == 1: selected_diagonals.append(2) # 13.5.1
                else: selected_diagonals.append(2) # 13.4
            else: # 13.5.2 tunnel
                selected_diagonals.extend([2, 0, 1, 2])

    points = []
    for diag in selected_diagonals:
        t = solve_trilinear_diagonal(vertex_values, diag, isovalue)
        if t is not None: points.append((t, diag))
    return points

def solve_trilinear_diagonal(vertex_values, diagonal_index, isovalue):
    """
    Solves f(t) = I along the specified diagonal.
    Returns t in [0, 1] or None if no intersection.
    """
    # Trilinear interpolation along a diagonal reduces to a cubic polynomial f(t) = at^3 + bt^2 + ct + d [G]
    # Coefficients are computed from vertex values and diagonal geometry.
    roots = solve_cubic_equation(vertex_values, diagonal_index, isovalue)
    for t in roots:
        if 0 <= t <= 1: return t
    return None
```

### Parameters
- **Cell Sign Configuration:** Integer index from 0 to 13 [S1].
- **Ambiguous Face Sign:** Determined by asymptotic decider [S2].
- **DeVella's Necklace Test Threshold:** Not given in the sources. [G] Typically involves checking the sign of the trilinear function at the center of the diagonal or comparing vertex signs with the function value at the midpoint.

### Pitfalls
- **Precision:** Evaluating the intersection point requires significant computational operations [S1]. Floating-point precision errors in solving the cubic equation may lead to missed intersections or incorrect topology [S1].
- **Tunnels:** The "one-point-per-disjoint-component" rule is relaxed for tunnels [S1][S2]. Sufficient points are needed to capture the complexity of the manifold [S1][S2].
- **Ambiguous Face Detection:** Incorrect identification of ambiguous faces or their signs leads to wrong diagonal selection [S2].

## Operating characteristics
- **Time Complexity:** $O(1)$ per voxel, as it involves a fixed number of case checks and cubic root solving [G]. However, solving the cubic equation is more expensive than linear interpolation used in standard MC [G].
- **Memory Complexity:** $O(1)$ per voxel, storing only the vertex values and selected diagonals [G].
- **Strengths:** Topologically accurate representation of the analytical isosurface [S1]. Handles ambiguous configurations correctly without lookup tables [S1][S2].
- **Limitations:** Computationally more expensive than linear MC due to cubic root solving [G]. Complex logic for diagonal selection [G].
- **When to use:** When topological accuracy is critical and the scalar field is well-approximated by trilinear interpolation [S1].
- **When not to use:** When performance is paramount and linear approximation is sufficient [G].

## Background
Originated from the need to accurately extract topologies of analytical isosurfaces defined by trilinear functions [S1]. The method analyzes the 31 cases from the exhaustive MC case-table (Chernyaev, 1995; Nielson, 2003; Lopes and Brodlie, 2003) [S1][S2]. It builds on the work of Nielson and Hamann (1991) on ambiguous faces and DeVella's necklace test [S1][S2].

## Sources
[S1] Topologically Accurate Dual Isosurfacing Using Ray Intersection, https://www.jvrb.org/jvrb/article/download/4.2007.4/html?inline=1, fetched 2026-10-29
[S2] Using Ray Intersection for Dual Isosurfacing, http://www.scitepress.net/api/PaperFile/Download?id=kJgq94qrpaA=&t=1&type=nHg6mGyw+oQ=&t=1&year=hwU2oyWbE78=&t=1, fetched 2026-10-29
