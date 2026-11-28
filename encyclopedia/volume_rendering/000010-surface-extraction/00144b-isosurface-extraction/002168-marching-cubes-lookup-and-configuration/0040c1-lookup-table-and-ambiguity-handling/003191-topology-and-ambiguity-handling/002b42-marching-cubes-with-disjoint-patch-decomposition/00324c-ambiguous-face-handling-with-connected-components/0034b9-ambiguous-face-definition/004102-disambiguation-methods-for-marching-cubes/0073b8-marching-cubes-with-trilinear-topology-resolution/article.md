# Marching Cubes with Trilinear Topology Resolution

Also called: Lopes' MC, Topologically Correct MC

## Overview
This method extends the standard Marching Cubes algorithm to accurately reproduce the topology of a piecewise trilinear interpolant by resolving topological ambiguities on cell faces and in the cell interior [S1]. Unlike the original algorithm, which ignores face and interior topology, or "implicit disambiguation" methods that consistently separate positive vertices, this approach examines the actual trilinear structure to avoid errors such as merging minima with saddles [S1].

## How it works
**Inputs:** A scalar field defined on a rectilinear grid (values at vertices) and an isovalue $I$.
**Outputs:** A triangulated isosurface mesh.

**Data Structures:**
*   **Grid:** A 3D array of scalar values $V(x,y,z)$.
*   **Lookup Table (LUT):** An extended table mapping vertex polarity configurations to triangle indices. The standard 256-entry LUT is insufficient; additional entries or logic are required to handle ambiguous cases [S1].
*   **Vertex Polarity:** A boolean classification for each vertex $v$:
    *   Negative: $V(v) < I$
    *   Positive: $V(v) > I$
    *   Critical: $V(v) = I$ (handled via specific logic, see pitfalls) [S1].

**Steps:**

1.  **Vertex Classification:** For each vertex in the grid, determine its polarity relative to the isovalue $I$. Vertices with values less than $I$ are negative; those greater are positive [S1].

2.  **Cell Classification:** For each voxel cell, determine the standard 256-case configuration based on the polarities of its 8 corner vertices.

3.  **Ambiguity Detection:** Identify cells that require topological resolution:
    *   **Ambiguous Faces:** A face of the cell is ambiguous if it has alternating vertex polarities (e.g., two adjacent vertices are positive, the other two are negative). This allows different choices for connecting edge intersection points, corresponding to hyperbolic arcs separating negative or positive vertices [S1].
    *   **Ambiguous Interior:** In the cell interior, opposite vertices can be separated by two isosurface sheets or connected by one isosurface. This occurs when the standard LUT case does not uniquely determine the topology due to the trilinear nature of the interpolant [S1].

4.  **Trilinear Analysis:** For cells with ambiguous faces or interior configurations, analyze the trilinear interpolant within the cell to determine the correct topology [S1].
    *   **Face Analysis:** Examine the bilinear interpolation on the ambiguous face. Determine if a saddle point exists within the face. A vertex is a local minimum if its four edge-connected neighbors have larger values. If a saddle exists, the standard linear interpolation assumption fails [S1].
    *   **Interior Analysis:** Determine if the isosurface consists of two separate sheets or a single connected component within the cell interior. This requires evaluating the trilinear function's behavior in the volume, not just on the edges [S1].

5.  **Topology Resolution:** Based on the trilinear analysis, select the correct triangulation from the possible ambiguous configurations. The LUT must be extended or modified to include these resolved cases [S1].

6.  **Mesh Generation:** Generate the triangles for the cell using the resolved configuration. Ensure connectivity with adjacent cells is maintained.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marching_cubes_trilinear(grid, isovalue):
    """
    Extract isosurface from grid using trilinear topology resolution.
    grid: 3D array of scalar values.
    isovalue: The threshold value for the isosurface.
    Returns: List of triangles (each triangle is a tuple of 3 vertex indices).
    """
    triangles = []
    rows, cols, depth = grid.shape
    
    # Precompute vertex polarities
    # polarity[r][c][d] is True if grid[r][c][d] > isovalue, False otherwise
    # Note: Equality isovalue handling requires special care [S1]
    polarity = grid > isovalue
    
    for r in range(rows - 1):
        for c in range(cols - 1):
            for d in range(depth - 1):
                # Get the 8 vertices of the current cell
                v = [
                    grid[r][c][d], grid[r+1][c][d], grid[r+1][c+1][d], grid[r][c+1][d],
                    grid[r][c][d+1], grid[r+1][c][d+1], grid[r+1][c+1][d+1], grid[r][c+1][d+1]
                ]
                p = [
                    polarity[r][c][d], polarity[r+1][c][d], polarity[r+1][c+1][d], polarity[r][c+1][d],
                    polarity[r][c][d+1], polarity[r+1][c][d+1], polarity[r+1][c+1][d+1], polarity[r][c+1][d+1]
                ]
                
                # Standard case index
                case_index = 0
                for i in range(8):
                    if p[i]:
                        case_index |= (1 << i)
                
                # Check for ambiguous faces
                # A face is ambiguous if it has alternating polarities
                # Faces are defined by 4 vertices. We check all 6 faces.
                # Face indices (from Lorensen & Cline):
                # 0: (0,1,2,3), 1: (4,5,6,7), 2: (0,1,5,4), 3: (2,3,7,6), 4: (0,3,7,4), 5: (1,2,6,5)
                faces = [
                    [0, 1, 2, 3], [4, 5, 6, 7],
                    [0, 1, 5, 4], [2, 3, 7, 6],
                    [0, 3, 7, 4], [1, 2, 6, 5]
                ]
                
                ambiguous_face = False
                for face in faces:
                    # Check if polarities alternate
                    # Alternating means not all same, and not two adjacent same and two adjacent same in a way that doesn't cross
                    # Specifically, for a square face, alternating means the diagonal pairs have different polarities
                    # e.g., p[0]!=p[2] and p[1]!=p[3] or p[0]!=p[1] and p[2]!=p[3] is not the definition.
                    # The definition is: "alternating vertex polarities" allowing different choices for connecting edge intersections.
                    # This corresponds to the case where the face has a saddle.
                    # In a bilinear patch, if the product of differences across diagonals is negative, it's ambiguous?
                    # Actually, the condition is that the face has one positive and one negative vertex on each diagonal?
                    # No, it's that the face has two positive and two negative vertices, and they are not adjacent.
                    # i.e., p[0] != p[2] and p[1] != p[3] (diagonals have different signs)
                    # OR p[0] != p[1] and p[2] != p[3] is not the right check.
                    # Let's use the standard check: sum of polarities is 2, and they are not adjacent.
                    # Adjacent pairs: (0,1), (1,2), (2,3), (3,0).
                    # If sum is 2, and no adjacent pair is both True, then it's alternating.
                    if sum([p[i] for i in face]) == 2:
                        # Check if adjacent vertices are same polarity
                        # If p[0]==p[1] or p[1]==p[2] or p[2]==p[3] or p[3]==p[0], then they are adjacent same.
                        # If none are adjacent same, then they are alternating (diagonal same).
                        if not (p[face[0]] == p[face[1]] or p[face[1]] == p[face[2]] or p[face[2]] == p[face[3]] or p[face[3]] == p[face[0]]):
                            ambiguous_face = True
                            break
                
                if ambiguous_face:
                    # Resolve ambiguity using trilinear analysis
                    # This involves checking the interior of the cell and the face
                    # For simplicity, we assume a resolved LUT entry or a specific function
                    # Lopes' method requires analyzing the trilinear function
                    # We will use a placeholder for the complex trilinear analysis
                    resolved_case = resolve_trilinear_ambiguity(v, p, case_index)
                    triangles.extend(get_triangles(resolved_case, v, isovalue))
                else:
                    # Standard case
                    triangles.extend(get_triangles(case_index, v, isovalue))
                    
    return triangles

def resolve_trilinear_ambiguity(v, p, case_index):
    """
    Determine the correct topology for an ambiguous cell using trilinear interpolation.
    This function analyzes the bilinear interpolation on faces and trilinear in the interior.
    """
    # Check for saddle points on ambiguous faces
    # A saddle exists if the bilinear interpolant has a critical point inside the face
    # This can be determined by checking the signs of the derivatives or the values at the center
    
    # For each ambiguous face, determine if the isosurface passes through it in a specific way
    # The key is to distinguish between two isosurface sheets and one connected sheet
    
    # Example logic for one face (simplified):
    # If the face has a saddle, the topology is determined by the interior configuration
    
    # This is a complex calculation involving solving for critical points of the trilinear function
    # For implementation, one might precompute the topology based on the relative values of the 8 vertices
    
    # Return the resolved case index
    # This depends on the specific implementation of Lopes' method
    # The source notes that small alterations were made to fix remaining errors
    return case_index # Placeholder: In reality, this returns a different case index if ambiguity is resolved

def get_triangles(case_index, v, isovalue):
    """
    Get triangle indices for a given case index.
    v: list of 8 vertex values
    isovalue: threshold
    """
    # Standard Marching Cubes LUT
    # This is a simplified representation
    luts = {
        0: [], 1: [], # ...
        # Full LUT would be here
    }
    
    # For ambiguous cases, the LUT entry must be the resolved one
    # This function assumes the case_index passed is the resolved one
    
    # Edge intersection points
    edges = [
        (0,1), (1,2), (2,3), (3,0), # Bottom face
        (4,5), (5,6), (6,7), (7,4), # Top face
        (0,4), (1,5), (2,6), (3,7)  # Vertical edges
    ]
    
    # Calculate intersection points on edges
    intersections = []
    for e in edges:
        v1, v2 = v[e[0]], v[e[1]]
        if (v1 - isovalue) * (v2 - isovalue) < 0:
            t = (isovalue - v1) / (v2 - v1)
            # Interpolate position (assuming unit cube for simplicity)
            # In reality, positions are needed
            intersections.append(t)
        else:
            intersections.append(None)
            
    # Return triangles based on case_index and intersections
    # This is a placeholder for the actual LUT lookup
    return []
```

**Parameters:**
*   **Isovalue:** The threshold for the isosurface. Critical isovalues (where a vertex value equals the isovalue) require special handling to ensure meaningful results [S1].
*   **Grid Resolution:** Higher resolution reduces discretization error but increases computational cost.

**Edge Cases and Pitfalls:**
*   **Critical Isovalues:** If a vertex value equals the isovalue, the polarity is undefined. The method must handle this to avoid topological errors. The source notes that critical isovalues detected by this method are only meaningful if a topologically correct scheme is used [S1].
*   **Saddle Points:** The presence of saddle points on faces or in the interior is the root cause of ambiguity. Failure to detect them leads to incorrect topology (e.g., merging minima with saddles) [S1].
*   **Precision:** Numerical precision issues in determining the sign of differences or the existence of saddle points can lead to errors.

## Operating characteristics
*   **Time Complexity:** Higher than standard Marching Cubes due to the additional trilinear analysis for ambiguous cells. The number of ambiguous cells depends on the data.
*   **Memory Complexity:** Requires an extended LUT or additional logic to handle ambiguous cases.
*   **Strengths:** Produces a topologically correct isosurface that matches the piecewise trilinear interpolant. Avoids errors like merging minima with saddles.
*   **Limitations:** More complex to implement than standard MC. Requires careful handling of critical isovalues.
*   **When to Use:** When topological correctness is critical, such as in scientific visualization where accurate representation of minima, maxima, and saddles is important.
*   **When Not to Use:** When performance is paramount and topological errors are acceptable, or when the data is smooth and unlikely to have ambiguous configurations.

## Background
The method is based on a technique presented by Lopes in his dissertation, with small alterations to fix remaining topology errors and handle critical isovalues [S1]. It addresses the limitations of the original Marching Cubes algorithm, which only uses linear interpolation along edges and ignores face and interior topology [S1]. It also differs from "implicit disambiguation" proposed by Montani et al., which consistently separates positive vertices on ambiguous faces but can lead to topological errors [S1].

## Sources
[S1] Exploring Scalar Fields Using Critical Isovalues, https://escholarship.org/content/qt8h5134ht/qt8h5134ht.pdf?t=ptt27g, fetched 2026-10-27
