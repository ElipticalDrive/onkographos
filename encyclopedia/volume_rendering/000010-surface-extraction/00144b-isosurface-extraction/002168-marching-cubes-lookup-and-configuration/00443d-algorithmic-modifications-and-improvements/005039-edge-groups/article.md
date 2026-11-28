# Edge Groups

Also called: Edge Group Analysis

## Overview
Edge Groups is a classification framework for isosurface extraction methods that groups triangle configurations by the connectivity of the cell edges intersected by the isosurface, enabling systematic analysis and bounding of triangle quality [S1]. It partitions edge configurations into equivalence classes based on cubic cell symmetries, where the quality of an edge group is defined by the worst-case triangle quality generated within that group [S1]. Unlike prior approaches that rely on intuition to move sampling grid points or cut points, edge groups provide a formal combinatorial framework that explains why warping or connectivity modifications improve quality [S1]. It serves as an analytical tool to identify and replace problematic triangulation cases in [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md) [S1].

## How it works
**Inputs and Outputs:**
* **Input:** The standard Marching Cubes (MC) cell configuration (case index derived from vertex signs relative to an isovalue) and the geometric positions of the 8 vertices of the cubic cell [S1].
* **Output:** A classification of each generated triangle into one of 8 Edge Groups, and optionally, a modified MC lookup table where connectivity for problematic cases is replaced to avoid low-quality edge groups [S1].

**Data Structures:**
* **Edge Group Index:** An integer from 0 to 7 identifying the equivalence class of the triangle's active edges [S1].
* **Cut Point Parameters:** A vector $\mathbf{t} = (t_A, t_B, t_C)$ where each $t_i \in [0, 1]$ represents the normalized position of a cut point along an active edge [S1].
* **Quality Function:** $q: \mathbb{R}^n \to \mathbb{R}$, where $n$ is the number of active edges in the specific MC case (ranging from 3 to 12) [S1].

**Definitions:**
* **Active Edge:** An edge of the cubic cell intersected by the isosurface [S1].
* **Cut Point:** The intersection point of the isosurface with an active edge [S1].
* **Edge Group:** A set of symmetrically equivalent edge configurations that define a triangle [S1].
* **Triangle Quality Metric:** Radii-ratio, defined as the ratio of the incircle radius to the circumcircle radius of a triangle, normalized to $[0, 1]$. An equilateral triangle achieves the maximum quality of 1 [S1].

**Numbered Steps:**

1. **Identify Active Edges:** For a given MC case, determine which of the 12 edges of the cubic cell are intersected by the isosurface [S1].
2. **Determine Triangle Connectivity:** Based on the MC case index, determine which three active edges form each triangle in the cell [S1].
3. **Classify into Edge Groups:** Map each triangle's set of three active edges to one of the 8 Edge Groups by identifying the geometric relationship between the three edges and applying cubic symmetries to find the canonical representative [S1]. There are exactly 8 distinct edge groups in MC [S1].
4. **Parameterize Cut Points:** For a triangle defined by edges $A, B, C$, define the cut points $c_A, c_B, c_C$ using parameters $t_A, t_B, t_C \in [0, 1]$ [S1]. The positions are calculated as:
   $$
   c_A = p_0 + t_A (p_3 - p_0)
   $$
   $$
   c_B = p_0 + t_B (p_1 - p_0)
   $$
   $$
   c_C = p_4 + t_C (p_5 - p_4)
   $$
   where the vertex indices follow the standard MC labeling convention for the specific edges [S1]. Generally, $t_i$ represents the normalized distance from one endpoint of the edge to the cut point [S1].
5. **Evaluate Quality Bounds:** For each Edge Group, analyze the function $q(\mathbf{t})$ over the parameter space $\Theta = [0, 1]^n$ (where $n$ is the number of active edges in the case) to find the minimum and maximum radii-ratio [S1]. This identifies the worst-case and best-case triangles for that group [S1].
6. **Identify Problematic Groups:** Analyze the histograms of triangle quality for each group [S1]. Edge Group 2 is identified as the dominant source of degenerate triangles (quality approaching 0) and has a non-zero density of such triangles [S1]. Groups 0, 1, 3, and 6 can also produce degenerate triangles but with lower density or different characteristics [S1]. Groups 4, 5, and 7 do not produce degenerate triangles and share the same lower bound quality [S1].
7. **Modify Lookup Table (Optional Improvement):** To improve mesh quality, replace the triangulation connectivity in MC cases that generate Edge Group 2 with an alternative triangulation that uses a different edge group (e.g., from Groups 4, 5, or 7) [S1]. This exchange replaces a high-probability bad triangle with a slightly worse but non-degenerate one, improving overall mesh quality by an order of magnitude [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_radii_ratio_quality(p1, p2, p3):
    """Computes the radii-ratio quality of a triangle defined by vertices p1, p2, p3."""
    a = math.sqrt((p2[0]-p3[0])**2 + (p2[1]-p3[1])**2 + (p2[2]-p3[2])**2)
    b = math.sqrt((p1[0]-p3[0])**2 + (p1[1]-p3[1])**2 + (p1[2]-p3[2])**2)
    c = math.sqrt((p1[0]-p2[0])**2 + (p1[1]-p2[1])**2 + (p1[2]-p2[2])**2)
    s = (a + b + c) / 2.0
    area_sq = s * (s - a) * (s - b) * (s - c)
    if area_sq <= 0: return 0.0
    area = math.sqrt(area_sq)
    r_in = area / s
    R_circ = (a * b * c) / (4.0 * area)
    if R_circ == 0: return 0.0
    return r_in / R_circ

def classify_triangle_into_edge_group(edge_indices):
    """
    Maps a triplet of active edge indices to one of 8 Edge Groups.
    edge_indices: tuple of 3 edge IDs (0-11).
    Returns: integer 0-7.
    """
    # Canonicalize edge configuration under cubic symmetries.
    # [G] Precompute equivalence classes of edge triplets under the 24 symmetries of the cube.
    # Return the group ID corresponding to the canonical representative.
    return 0

def analyze_edge_group_quality(group_id, active_edges, cube_vertices):
    """
    Analyzes the quality bounds for a specific Edge Group by sampling the parameter space.
    """
    min_q, max_q = 1.0, 0.0
    step = 0.1
    # Map group-specific edge indices to cube vertex pairs for cut point calculation
    edge_pairs = get_edge_vertex_pairs(group_id)
    for tA in range(0, 11):
        for tB in range(0, 11):
            for tC in range(0, 11):
                ta, tb, tc = tA/10.0, tB/10.0, tC/10.0
                cA = [cube_vertices[edge_pairs[0][0]][i] + ta * (cube_vertices[edge_pairs[0][1]][i] - cube_vertices[edge_pairs[0][0]][i]) for i in range(3)]
                cB = [cube_vertices[edge_pairs[1][0]][i] + tb * (cube_vertices[edge_pairs[1][1]][i] - cube_vertices[edge_pairs[1][0]][i]) for i in range(3)]
                cC = [cube_vertices[edge_pairs[2][0]][i] + tc * (cube_vertices[edge_pairs[2][1]][i] - cube_vertices[edge_pairs[2][0]][i]) for i in range(3)]
                q = compute_radii_ratio_quality(cA, cB, cC)
                if q < min_q: min_q = q
                if q > max_q: max_q = q
    return min_q, max_q

def get_edge_vertex_pairs(group_id):
    """Returns the vertex indices for the canonical edges of a given edge group."""
    # [G] Lookup table mapping group IDs to canonical edge vertex pairs based on MC labeling.
    return [(0, 3), (0, 1), (4, 5)]

def improve_mc_table(mc_table):
    """
    Modifies the MC lookup table to reduce degenerate triangles by replacing Edge Group 2.
    """
    improved_table = [list(case) for case in mc_table]
    for case_index in range(256):
        triangles = improved_table[case_index]
        for tri_idx, triangle in enumerate(triangles):
            group = classify_triangle_into_edge_group(triangle['edges'])
            if group == 2:
                # Replace connectivity with an alternative that avoids Edge Group 2.
                # [G] Swap triangle['edges'] with a known safe configuration for this case.
                improved_table[case_index][tri_idx] = {'edges': triangle['edges']}
    return improved_table
```

**Parameters:**
* **Cut point position parameter $t_i$:** A scalar value in the range $[0, 1]$ representing the normalized position of the $i$-th cut point along an active cell edge [S1].
* **Isosurface Value:** Determines which edges are active [G].

**Edge Cases and Pitfalls:**
* **Degenerate Triangles:** Edge Groups 0, 1, 2, 3, and 6 can produce degenerate triangles (quality 0). Edge Group 2 is the most problematic, having a non-zero density of degenerate triangles [S1].
* **Symmetry Handling:** Correctly identifying edge groups requires careful handling of cubic cell symmetries [S1].
* **Connectivity Consistency:** When modifying the MC table to avoid Edge Group 2, care must be taken to ensure that the new connectivity does not introduce non-manifold edges or holes in the mesh [G].

## Operating characteristics
* **Time Complexity:** Classification is $O(1)$ per triangle with a precomputed lookup table [G]. Quality bound analysis is computationally expensive and performed offline [G].
* **Memory Complexity:** Requires minimal additional memory for group classification [G]. The modified MC table uses the same memory as the standard table [G].
* **Measured Performance:** Replacing Edge Group 2 in MC cases improves overall mesh quality by an order of magnitude in most tested datasets [S1].
* **Strengths:** Provides a formal combinatorial framework for understanding and improving mesh quality [S1]. Identifies Edge Group 2 as the dominant source of degenerate triangles [S1].
* **Limitations:** The improvement trades a high-probability bad triangle for a slightly worse but non-degenerate one [S1].

## Background
Edge Groups was introduced by Carlos A. Dietrich, João L. D. Comba, Luciana P. Nedel, Carlos E. Scheidegger, and Cláudio T. Silva in the paper "Edge Groups: An Approach to Understanding the Mesh Quality of Marching Methods" [S1]. The paper was received on March 31, 2008, and accepted on August 11, 2008. It was published in the IEEE Transactions on Visualization and Computer Graphics. The work builds on the foundational Marching Cubes algorithm and aims to provide a theoretical basis for the empirical improvements seen in methods like Macet, which warps grid points to improve quality [S1].

## Sources
[S1] Edge Groups: An Approach to Understanding the Mesh Quality of Marching Methods, https://doi.org/10.1109/tvcg.2008.122, fetched 2026-10-26
