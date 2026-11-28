# Naive Tetrahedral Surfacing

Also called: Naive Tetrahedral Marching Tetrahedra, Pairwise Intersection Method, Tetrahedral Marching Tetrahedra (Naive/Brute-Force Variant)

## Overview
Naive Tetrahedral Surfacing is an isosurface extraction method that processes each tetrahedron independently to generate a collection of individual, unconnected polygons [S1, unofficial]. It differs from standard Marching Tetrahedra by iterating through all pairs of sample points to find intersections with an isosurface, rather than using the standard 14-case lookup table based on vertex signs [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A tetrahedron defined by four sample points (vertices) and their corresponding scalar field values [S1, unofficial].
- Input: `iso_val`, the scalar threshold defining the isosurface [S1, unofficial].
- Output: A list of 3D vertices (`vertices`) representing the intersection points on the tetrahedron edges [S1, unofficial].

**Data Structures:**
- `samp_pts`: A list of 4 `Vector3` objects representing the tetrahedron vertices [S1, unofficial].
- `surf_vals`: A list of 4 `float` values representing the scalar field evaluated at each vertex in `samp_pts` [S1, unofficial].
- `less_than_iso`: A list of `Vector3` objects containing vertices where `value < iso_val` [S1, unofficial].
- `less_val`: A list of `float` values corresponding to `less_than_iso` [S1, unofficial].
- `more_than_iso`: A list of `Vector3` objects containing vertices where `value >= iso_val` [S1, unofficial].
- `more_val`: A list of `float` values corresponding to `more_than_iso` [S1, unofficial].
- `vertices`: An output list of `Vector3` objects storing the calculated intersection points [S1, unofficial].

**Steps:**
1. Evaluate the scalar function at all sample points (vertices) of the tetrahedron to obtain `surf_vals` [S1, unofficial].
2. Partition the sample points and their values into two lists: `less_than_iso` and `less_val` for points where `value < iso_val`, and `more_than_iso` and `more_val` for points where `value >= iso_val` [S1, unofficial].
3. Iterate through every point `v1` in `less_than_iso` and every point `v2` in `more_than_iso` to form candidate edges [S1, unofficial].
4. For each pair `(v1, v2)`, calculate the intersection vertex `new_vertex` using linear interpolation [S1, unofficial].
5. Add `new_vertex` to the output `vertices` list. If the index `i` of the `more_than_iso` point is even, push to front; otherwise, push to back [S1, unofficial]. This ordering logic is specific to this implementation's output format and does not guarantee consistent triangle winding or connectivity without further processing [S1, unofficial].

**Formula:**
The intersection vertex is calculated as:
$$ \text{new\_vertex} = \frac{v1 \cdot \text{diff}_2 + v2 \cdot \text{diff}_1}{\text{diff\_tot}} $$
Where:
- $\text{diff\_tot} = |val1 - val2|$ [S1, unofficial]
- $\text{diff}_1 = |val1 - \text{iso\_val}|$ [S1, unofficial]
- $\text{diff}_2 = |val2 - \text{iso\_val}|$ [S1, unofficial]

This is equivalent to standard linear interpolation:
$$ \text{new\_vertex} = v1 + (v2 - v1) \cdot \frac{\text{iso\_val} - val1}{val2 - val1} $$ [S1, unofficial]

**Parameters:**
- `iso_val`: The scalar threshold value defining the isosurface. Typical value depends on the field (e.g., 0 for signed distance fields) [S1, unofficial].
- `eps`: A hardcoded epsilon value (0.1) used in `less_than` for lexicographical comparison of points, though the code comments note this function was abandoned because duplicate vertices are considered acceptable [S1, unofficial].

**Pitfalls:**
- The algorithm performs a pairwise comparison of all 'less' and 'more' points [S1, unofficial]. For a tetrahedron with 4 vertices, if 2 are less and 2 are more, this results in $2 \times 2 = 4$ intersection calculations, whereas standard MT only generates 2 edges [S1, unofficial]. This is computationally inefficient and may generate redundant or incorrect geometry if the topology assumption (that every less point connects to every more point) is invalid for the specific scalar field distribution [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
from typing import List

def naive_tetrahedral_surfacing(samp_pts: List[Vector3], surf_vals: List[float], iso_val: float) -> List[Vector3]:
    """
    Extracts isosurface intersection vertices from a single tetrahedron using naive pairwise intersection.
    
    Args:
        samp_pts: List of 4 Vector3 vertices of the tetrahedron.
        surf_vals: List of 4 floats, scalar values at each vertex.
        iso_val: The isosurface threshold.
        
    Returns:
        List of Vector3 intersection points.
    """
    less_than_iso: List[Vector3] = []
    less_val: List[float] = []
    more_than_iso: List[Vector3] = []
    more_val: List[float] = []
    vertices: List[Vector3] = []
    
    # Step 2: Partition points
    for i in range(4):
        if surf_vals[i] < iso_val:
            less_than_iso.append(samp_pts[i])
            less_val.append(surf_vals[i])
        else:
            more_than_iso.append(samp_pts[i])
            more_val.append(surf_vals[i])
            
    # Step 3 & 4: Pairwise intersection
    for i, v1 in enumerate(less_than_iso):
        val1 = less_val[i]
        for j, v2 in enumerate(more_than_iso):
            val2 = more_val[j]
            
            diff_tot = abs(val1 - val2)
            # Guard against division by zero if values are identical [G]
            if diff_tot == 0:
                continue
                
            diff_1 = abs(val1 - iso_val)
            diff_2 = abs(val2 - iso_val)
            
            # Formula: new_vertex = (v1 * diff_2 + v2 * diff_1) / diff_tot
            new_vertex = (v1 * diff_2 + v2 * diff_1) / diff_tot
            
            # Step 5: Add to output with specific ordering
            # If the index of the more_than_iso point is even, push to front; otherwise, push to back [S1, unofficial]
            if j % 2 == 0:
                vertices.insert(0, new_vertex)
            else:
                vertices.append(new_vertex)
                
    return vertices
```

## Operating characteristics
- **Time Complexity:** $O(1)$ per tetrahedron, but with a higher constant factor than standard MT [S1, unofficial]. In the worst case (2 less, 2 more vertices), it performs 4 intersection calculations instead of 2 [S1, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary space per tetrahedron for storing partitioned lists [S1, unofficial].
- **Strengths:** Simple to implement; does not require a complex case lookup table [S1, unofficial].
- **Limitations:** Generates redundant vertices; does not produce connected polygons directly; computationally inefficient for certain configurations; lacks topological consistency guarantees [S1, unofficial].
- **When to use:** When simplicity of implementation is prioritized over performance and connectivity, or for educational purposes [S1, unofficial].
- **When not to use:** When high performance, mesh connectivity, or topological correctness is required [S1, unofficial].

## Background
Naive Tetrahedral Surfacing is an unofficial method described in a GitHub repository [S1, unofficial]. It represents a brute-force variant of the Marching Tetrahedra algorithm, abandoning the standard case-based approach in favor of pairwise edge intersection [S1, unofficial]. The specific implementation details, including the epsilon value and vertex ordering logic, are derived from this source [S1, unofficial].

## Sources
[S1] ajinkyajayawant/surfacing-tetrahedral, https://github.com/ajinkyajayawant/surfacing-tetrahedral, fetched 2026-10-27 (unofficial)
