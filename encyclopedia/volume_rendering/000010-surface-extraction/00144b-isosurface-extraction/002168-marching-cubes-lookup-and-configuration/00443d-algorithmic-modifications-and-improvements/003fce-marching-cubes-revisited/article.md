# Marching Cubes Revisited

Also called: Modified Marching Cubes

## Overview
Marching Cubes Revisited is a topologically correct variant of the Marching Cubes algorithm designed to eliminate surface holes and ambiguities inherent in the original Lorensen and Cline formulation [S1]. It achieves this by enforcing an interval condition on the implicit function's gradient within each grid cell, which restricts the possible vertex sign configurations to 9 valid cases and resolves the remaining topological ambiguities in cases 4 and 6 through specific geometric constraints [S1].

## How it works
**Inputs:**
*   $F: \mathbb{R}^3 \to \mathbb{R}$: A smooth implicit function defining the surface $S = F^{-1}(0)$ [S1].
*   Grid: A uniform cubic grid.
*   Isovalue: 0 [S1].

**Outputs:**
*   A triangle mesh approximating the surface $S$ [S1].

**Data Structures:**
*   **Grid Cell:** A cube defined by 8 vertices. Each vertex $v_i$ has a sign $s_i = \text{sign}(F(v_i))$.
*   **Case Index:** An integer derived from the 8 vertex signs, identifying the intersection configuration.
*   **Edge Midpoints:** For each intersected edge, a vertex is placed at the geometric center of the edge [S1].

**Assumptions and Constraints:**
1.  **Regular Value:** 0 is a regular value of $F$, meaning $\nabla F \neq 0$ at every point on $S$ [S1].
2.  **Vertex Exclusion:** $S$ does not contain any grid vertices [S1].
3.  **Single Intersection:** $S$ intersects each grid edge at most once [S1].
4.  **Interval Condition:** For every cell $C$, one of the following must hold:
    $$0 \notin \Box F(C) \quad \vee \quad 0 \notin \langle \Box \nabla F(C), \Box \nabla F(C) \rangle$$
    where $\Box F(C)$ is the range of $F$ over $C$, and $\Box \nabla F(C)$ is the range of the gradient. The second clause implies that for all $x, y \in C$:
    $$\langle \nabla F(x), \nabla F(y) \rangle > 0$$
    This ensures the gradient direction does not flip within the cell, guaranteeing the surface is parametrizable along at least one axis [S1].

**Algorithm Steps:**
1.  **Evaluate Signs:** For each grid cell, compute the sign of $F$ at all 8 vertices.
2.  **Check Interval Condition:** Verify if the cell satisfies the interval condition. If not, the method's topological guarantees do not apply (though the source notes this is a constraint for the *validity* of the reduced case set, the algorithm proceeds by assuming valid inputs or handling invalid cases via the octree refinement mentioned in the overview) [S1].
3.  **Determine Case Index:** Map the 8 vertex signs to one of the 14 possible configurations (up to rotation, mirroring, and sign change) [S1].
4.  **Filter Invalid Cases:** Discard configurations 5, 8, 12, 13, and 14. These are impossible under the interval condition because they require the gradient to change direction or fail the inner product condition [S1].
5.  **Triangulate Valid Cases:** For the remaining 9 configurations, generate triangles using edge midpoints:
    *   **Standard Cases:** Connect edge midpoints according to the standard connectivity for the specific case index.
    *   **Ambiguous Cases (4 and 6):** These cases have a single face where the function changes sign along all four edges, resulting in four potential vertices. To resolve the ambiguity, connect the vertices such that the resulting segments are parallel to $(1,1,0)$, $(1,0,1)$, or $(0,1,1)$, depending on the orientation of the ambiguous face [S1].
6.  **Refinement (Optional):** For higher accuracy, the algorithm can be extended using a balanced octree to remove the initial grid constraints, but the core topological correctness relies on the steps above [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def march_cubes_revisited(grid, F):
    """
    grid: 3D array of shape (Nx, Ny, Nz) containing vertex positions or indices.
    F: Function returning scalar value at a 3D point.
    Returns: List of triangles (each triangle is a tuple of 3 vertex indices).
    """
    triangles = []
    
    # Predefined connectivity for the 9 valid cases (excluding 5,8,12,13,14)
    # Case 4 and 6 require special handling for ambiguity.
    # Standard MC lookup tables are modified to exclude invalid cases and define
    # specific connections for 4 and 6 based on face orientation.
    
    for each cell in grid:
        # 1. Evaluate signs
        signs = []
        for vertex in cell.vertices:
            val = F(vertex)
            if val > 0:
                signs.append(1)
            else:
                signs.append(-1) # Assuming 0 is not on vertex per assumption
        
        # 2. Compute case index (bitmask of signs)
        case_index = compute_case_index(signs)
        
        # 3. Check for invalid configurations
        if case_index in [5, 8, 12, 13, 14]:
            # These configurations violate the interval condition.
            # In a strict implementation, this indicates the input F does not
            # satisfy the assumptions for this method.
            continue 
            
        # 4. Triangulate
        if case_index == 4:
            # Ambiguous case: resolve based on face orientation
            # Identify the face with sign changes on all 4 edges.
            # Connect vertices such that segments are parallel to (1,1,0), (1,0,1), or (0,1,1).
            triangles.extend(resolve_ambiguity_case_4(cell, signs))
        elif case_index == 6:
            # Ambiguous case: resolve based on face orientation
            triangles.extend(resolve_ambiguity_case_6(cell, signs))
        else:
            # Standard triangulation for the other 7 valid cases
            triangles.extend(triangulate_standard_case(case_index, cell))
            
    return triangles

def resolve_ambiguity_case_4(cell, signs):
    """
    Case 4 has a specific ambiguous face.
    The rule is to connect vertices such that the segment is parallel to
    (1,1,0), (1,0,1), or (0,1,1) depending on the face normal/orientation.
    """
    # Identify the ambiguous face (e.g., xy, xz, or yz plane)
    # Based on the face orientation, select the diagonal connection.
    # Example: If ambiguous face is normal to z (xy plane), connect parallel to (1,1,0) or (1,-1,0)?
    # Source specifies (1,1,0), (1,0,1), (0,1,1). These correspond to diagonals on xy, xz, yz faces.
    # We must determine which diagonal is chosen. The source implies a deterministic choice
    # based on the face orientation to ensure topological correctness.
    pass # Implementation detail depends on exact face identification logic.

def resolve_ambiguity_case_6(cell, signs):
    """
    Similar to Case 4, resolve ambiguity for Case 6.
    """
    pass
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
