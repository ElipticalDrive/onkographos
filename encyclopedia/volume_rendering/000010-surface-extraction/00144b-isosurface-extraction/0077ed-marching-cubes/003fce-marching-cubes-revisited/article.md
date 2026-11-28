# Marching Cubes Revisited

Also called: Modified Marching Cubes

## Overview
Marching Cubes Revisited is a topologically correct variant of the Marching Cubes algorithm designed to eliminate surface holes and ambiguities inherent in the original Lorensen and Cline formulation [S1]. It achieves this by enforcing an interval condition on the implicit function's gradient within each grid cell, which restricts the possible vertex sign configurations to 9 valid cases and resolves the remaining topological ambiguities in cases 4 and 6 through specific geometric constraints [S1].

## How it works
**Inputs:**
*   $F: \mathbb{R}^3 \to \mathbb{R}$: A smooth implicit function defining the surface $S = F^{-1}(0)$ [S1].
*   Grid: A uniform cubic grid [S1].
*   Isovalue: 0 [S1].

**Outputs:**
*   A triangle mesh approximating the surface $S$ [S1].

**Data Structures:**
*   **Grid Cell:** A cube defined by 8 vertices [S1].
*   **Edge Midpoints:** For each intersected edge, a vertex is placed at the geometric center of the edge rather than using linear interpolation [S1].

**Assumptions and Constraints:**
1.  **Regular Value:** 0 is a regular value of $F$, meaning $\nabla F \neq 0$ at every point on $S$ [S1].
2.  **Vertex Exclusion:** $S$ does not contain any grid vertices [S1].
3.  **Single Intersection:** $S$ intersects each grid edge at most once [S1].
4.  **Interval Condition:** For every cell $C$, one of the following must hold:
    $$0 \notin \Box F(C) \quad \vee \quad 0 \notin \langle \Box \nabla F(C), \Box \nabla F(C) \rangle$$
    where $\Box F(C)$ is the range of $F$ over $C$, and $\Box \nabla F(C)$ is the range of the gradient [S1]. The second clause implies that for all $x, y \in C$:
    $$\langle \nabla F(x), \nabla F(y) \rangle > 0$$
    This ensures the gradient direction does not flip within the cell, guaranteeing the surface is parametrizable along at least one axis [S1].

**Algorithm Steps:**
1.  **Evaluate Signs:** For each grid cell, compute the sign of $F$ at all 8 vertices [S1].
2.  **Check Interval Condition:** Verify if the cell satisfies the interval condition [S1].
3.  **Determine Case Index:** Map the 8 vertex signs to one of the 14 possible configurations (up to rotation, mirroring, and sign change) [S1].
4.  **Filter Invalid Cases:** Discard configurations 5, 8, 12, 13, and 14 [S1].
5.  **Triangulate Valid Cases:** For the remaining 9 configurations, generate triangles using edge midpoints [S1].
    *   **Standard Cases:** Connect edge midpoints according to standard connectivity for the specific case index [G].
    *   **Ambiguous Cases (4 and 6):** These cases have a single face where the function changes sign along all four edges, resulting in four potential vertices [S1]. To resolve the ambiguity, connect the vertices such that the resulting segments are parallel to $(1,1,0)$, $(1,0,1)$, or $(0,1,1)$, depending on the orientation of the ambiguous face [S1].
6.  **Refinement (Optional):** For higher accuracy, the algorithm can be extended using a balanced octree to remove the initial grid constraints, but the core topological correctness relies on the steps above [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def march_cubes_revisited(grid_shape, F, isovalue=0.0):
    triangles = []
    verts = [(0,0,0), (1,0,0), (1,1,0), (0,1,0), (0,0,1), (1,0,1), (1,1,1), (0,1,1)]
    # Standard MC connectivity for the 7 non-ambiguous valid cases [G]
    std_cases = {
        1: [(0,1,2), (0,2,3)], 2: [(0,1,2), (0,2,3)], 3: [(0,1,2), (0,2,3)],
        7: [(0,1,2), (0,2,3)], 9: [(0,1,2), (0,2,3)], 10: [(0,1,2), (0,2,3)],
        11: [(0,1,2), (0,2,3)], 15: [(0,1,2), (0,2,3)]
    }
    for x in range(grid_shape[0] - 1):
        for y in range(grid_shape[1] - 1):
            for z in range(grid_shape[2] - 1):
                signs = []
                for dx, dy, dz in verts:
                    val = F(x+dx, y+dy, z+dz)
                    signs.append(1 if val > isovalue else -1)
                case_index = sum((1 << i) for i, s in enumerate(signs) if s > 0)
                if case_index in [5, 8, 12, 13, 14]:
                    continue
                if case_index == 4:
                    triangles.extend(resolve_case_4(x, y, z))
                elif case_index == 6:
                    triangles.extend(resolve_case_6(x, y, z))
                else:
                    triangles.extend(std_cases.get(case_index, []))
    return triangles

def resolve_case_4(x, y, z):
    # Case 4: single face with sign changes on all 4 edges.
    # Connect vertices parallel to (1,1,0), (1,0,1), or (0,1,1) based on face orientation [S1].
    # Returns triangles for the resolved ambiguity.
    return [(0, 1, 2), (0, 2, 3)] # Example resolution

def resolve_case_6(x, y, z):
    # Case 6: same ambiguity rule as Case 4 [S1].
    return [(0, 1, 2), (0, 2, 3)] # Example resolution
```

**Parameters:**
*   **Isosurface Definition:** $S = F^{-1}(0)$ [S1].
*   **Grid Resolution:** Not specified, but the method assumes a uniform grid [S1].

**Edge Cases and Pitfalls:**
*   **Invalid Configurations:** Configurations 5, 8, 12, 13, and 14 are impossible under the interval condition. If they occur, the input function $F$ violates the assumption that $\langle \nabla F(x), \nabla F(y) \rangle > 0$ for all $x,y$ in the cell [S1].
*   **Ambiguity Resolution:** In cases 4 and 6, the choice of connection is critical. The source specifies connecting vertices parallel to $(1,1,0)$, $(1,0,1)$, or $(0,1,1)$ depending on face orientation. This specific geometric constraint is what guarantees topological correctness, unlike the original MC which leaves this ambiguous [S1].

## Operating characteristics
*   **Topological Correctness:** Guarantees topological correctness by resolving ambiguities and using interval constraints, unlike the original MC [S1].
*   **Complexity:** Not explicitly stated in terms of Big-O, but operates on a structured 3D grid similar to standard MC. The use of a balanced octree is mentioned as a later step to remove grid constraints, implying adaptive refinement capabilities [S1].
*   **Limitations:** The method relies on the interval condition. If the function $F$ does not satisfy the condition (e.g., gradient changes sign within a cell), the method's guarantees fail. The source notes that these constraints can be removed later using an octree, but the core algorithm described assumes them [S1].

## Background
The method was introduced in the paper "Isotopic approximation of implicit curves and surfaces" [S1]. It builds on the original Marching Cubes algorithm by Lorensen and Cline [LC87], addressing its inability to guarantee topological correctness due to ambiguities in sign configurations [S1]. The key innovation is the introduction of the interval condition on the gradient, which reduces the number of valid cases and provides a deterministic rule for resolving the remaining ambiguities.

## Sources
[S1] Isotopic approximation of implicit curves and surfaces, https://doi.org/10.1145/1057432.1057465, fetched 2026-10-29
