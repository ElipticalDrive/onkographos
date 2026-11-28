# Linear Sampling Method

Also called: 1D/2D Inverse Distance Interpolation for Tetrahedra, Linear Interpolation in Tetrahedral Cells

## Overview
The Linear Sampling Method is an accelerated interpolation technique for estimating scalar values within tetrahedral cells during direct volume ray casting [S1]. It exploits the linearity of scalar change in any direction within a tetrahedron to avoid expensive repeated 3D inverse distance interpolations [S1]. It sets itself apart from the conventional 3D inverse distance method by decomposing the 3D estimation into cheaper 2D and 1D operations, leveraging intersection test results to minimize computational overhead [S1].

## How it works
**Inputs and Outputs:**
- Input: A tetrahedral cell defined by four vertices $V_0, V_1, V_2, V_3$ with associated scalar values $S_0, S_1, S_2, S_3$. A ray segment passing through the cell, defined by entry point $P_{in}$ and exit point $P_{out}$, and a target sampling point $P$ on this segment.
- Output: The estimated scalar value $S(P)$ at point $P$.

**Data Structures:**
The tetrahedral cell data is stored in arrays containing vertex coordinates and scalar values [S1]. The specific layout is not detailed beyond "two arrays" in the source, but standard implementation requires access to vertex positions and scalars for the four corners and the three vertices of the two triangular faces intersected by the ray [S1].

**Method Steps:**
1.  **Identify Intersected Faces:** Determine which two triangular faces of the tetrahedron the ray enters and exits. Let the entry face be $F_{in}$ with vertices $\{A, B, C\}$ and the exit face be $F_{out}$ with vertices $\{D, E, F\}$ [S1]. Note: The vertices of the tetrahedron are partitioned such that $F_{in}$ and $F_{out}$ share an edge, and the union of their vertices constitutes the four tetrahedron vertices.
2.  **Estimate Entry Scalar ($S_{in}$):** Perform 2D inverse distance interpolation using the scalar values at vertices $A, B, C$ and the position $P_{in}$ to estimate the scalar value at the entry point [S1].
3.  **Estimate Exit Scalar ($S_{out}$):** Perform 2D inverse distance interpolation using the scalar values at vertices $D, E, F$ and the position $P_{out}$ to estimate the scalar value at the exit point [S1].
4.  **Estimate Point Scalar ($S(P)$):** Perform 1D inverse distance interpolation along the line segment connecting $P_{in}$ and $P_{out}$ using the estimated scalars $S_{in}$ and $S_{out}$ and the position of $P$ to estimate the scalar value at $P$ [S1].

**Formulas:**
The method relies on Inverse Distance Weighted (IDW) interpolation. For a set of $n$ known points $V_i$ with values $S_i$ and a query point $P$, the IDW estimate is:
$$ S(P) = \frac{\sum_{i=1}^{n} \frac{S_i}{d_i^k}}{\sum_{i=1}^{n} \frac{1}{d_i^k}} $$
where $d_i = \| P - V_i \|$ is the Euclidean distance, and $k$ is the power parameter (typically $k=2$ for standard IDW, though the source does not explicitly specify $k$, it refers to "inverse distance interpolation" [S1]).

For the **2D step** (Face Interpolation):
Given face vertices $V_1, V_2, V_3$ with scalars $S_1, S_2, S_3$ and intersection point $P_{face}$:
$$ S_{face} = \frac{\sum_{j=1}^{3} \frac{S_j}{\| P_{face} - V_j \|^k}}{\sum_{j=1}^{3} \frac{1}{\| P_{face} - V_j \|^k}} $$

For the **1D step** (Line Interpolation):
Given entry point $P_{in}$ with scalar $S_{in}$, exit point $P_{out}$ with scalar $S_{out}$, and target point $P$ on the segment:
$$ S(P) = \frac{\frac{S_{in}}{\| P - P_{in} \|} + \frac{S_{out}}{\| P - P_{out} \|}}{\frac{1}{\| P - P_{in} \|} + \frac{1}{\| P - P_{out} \|}} $$
Note: In 1D, if $k=1$ is assumed for "inverse distance", the formula simplifies. If $k=2$, squares are used. The source does not specify the power $k$ for the 1D/2D steps, only that it uses "inverse distance interpolation" [S1]. Standard practice often uses $k=2$.

**Parameters:**
- Sampling interval $D_t$: The method's performance relative to the conventional method depends on $D_t$ [S1].
- Power $k$: Not given in the sources.

**Edge Cases and Pitfalls:**
- **Precision:** Division by zero occurs if the sampling point $P$ coincides exactly with $P_{in}$ or $P_{out}$. Guard clauses are required [G].
- **Degenerate Input:** If the ray is tangent to a face or passes through a vertex, the distance calculation may become unstable. The source notes that values are calculated at low cost by utilizing intersection test results, implying robust intersection logic is a prerequisite [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def linear_sampling_method(tetrahedron, ray_entry, ray_exit, point_p, k=2):
    """
    Estimate scalar at point_p inside a tetrahedron using Linear Sampling Method.
    
    Args:
        tetrahedron: Object containing vertices [V0, V1, V2, V3] and scalars [S1].
        ray_entry: Point P_in where ray enters the tetrahedron.
        ray_exit: Point P_out where ray exits the tetrahedron.
        point_p: Point P where scalar is estimated.
        k: Power for inverse distance weighting.
        
    Returns:
        Estimated scalar value.
    """
    # 1. Identify intersected faces
    # The source states the method uses 2D IDW on the three vertices of the respective triangular faces.
    # We assume the ray entry/exit points lie on specific faces defined by the tetrahedron geometry.
    # This requires prior intersection tests to determine which faces are entered/exited.
    # Let's assume we have identified the entry face vertices and exit face vertices.
    
    # Example: Assume entry face is V0, V1, V2 and exit face is V0, V1, V3 (sharing edge V0-V1)
    # This mapping depends on the specific ray-tetrahedron intersection algorithm.
    # The source does not provide the face identification logic, only that it uses the faces [S1].
    
    entry_face_vertices = [tetrahedron.V0, tetrahedron.V1, tetrahedron.V2]
    entry_face_scalars = [tetrahedron.S0, tetrahedron.S1, tetrahedron.S2]
    
    exit_face_vertices = [tetrahedron.V0, tetrahedron.V1, tetrahedron.V3]
    exit_face_scalars = [tetrahedron.S0, tetrahedron.S1, tetrahedron.S3]
    
    # 2. Estimate scalar at entry point (2D IDW)
    s_in = inverse_distance_weighted(entry_face_vertices, entry_face_scalars, ray_entry, k)
    
    # 3. Estimate scalar at exit point (2D IDW)
    s_out = inverse_distance_weighted(exit_face_vertices, exit_face_scalars, ray_exit, k)
    
    # 4. Estimate scalar at point P (1D IDW)
    # Distance from P to entry and exit
    dist_in = distance(point_p, ray_entry)
    dist_out = distance(point_p, ray_exit)
    
    # Guard against division by zero
    if dist_in < 1e-9:
        return s_in
    if dist_out < 1e-9:
        return s_out
        
    # 1D IDW formula
    weight_in = 1.0 / (dist_in ** k)
    weight_out = 1.0 / (dist_out ** k)
    
    s_p = (s_in * weight_in + s_out * weight_out) / (weight_in + weight_out)
    
    return s_p

def inverse_distance_weighted(vertices, scalars, point, k):
    """
    Standard 2D/3D IDW interpolation.
    """
    numerator = 0.0
    denominator = 0.0
    
    for v, s in zip(vertices, scalars):
        d = distance(point, v)
        if d < 1e-9:
            return s # Exact match
        w = 1.0 / (d ** k)
        numerator += s * w
        denominator += w
        
    return numerator / denominator

def distance(p1, p2):
    return ((p1.x - p2.x)**2 + (p1.y - p2.y)**2 + (p1.z - p2.z)**2)**0.5
```

## Subtopics

### Inverse Distance Weighted Interpolation
The base scheme used for the 2D and 1D steps. It estimates values as a weighted sum of nearby points with influence decreasing as distance increases. Choose this when smooth interpolation from scattered data is needed. [Link](2336-inverse-distance-weighted-interpolation.md)

## Operating characteristics
- **Time Complexity**: The method reduces the complexity of interpolation inside a tetrahedron from a full 3D IDW calculation to two 2D IDW calculations and one 1D IDW calculation. This is faster than the conventional 3D IDW method [S1].
- **Memory Complexity**: No additional memory structures are required beyond the standard tetrahedral cell data (vertices and scalars) [S1].
- **Measured Performance**: The method is faster than the conventional method for equi-distant sampling schemas only when the sampling interval $D_t$ is small [S1]. It may perform worse than the conventional method for mid-point sampling schemas in theory, but performs better in practice because the values needed are calculated at very low cost by utilizing the results of computations performed during intersection tests [S1].
- **Strengths**: Significant speedup for dense sampling (small $D_t$) due to reuse of intersection data.
- **Limitations**: Performance gain is conditional on sampling density. For sparse sampling or mid-point schemas, the overhead of multiple IDW calls might negate benefits if intersection data is not efficiently reused.
- **When to use**: Use when high-frequency sampling (small $D_t$) is required within tetrahedral volumes and intersection tests are already being performed.
- **When not to use**: Use with caution for very sparse sampling where the overhead of 2D/1D calls outweighs the benefit of avoiding a single 3D call.

## Background
The Linear Sampling Method was introduced in the context of direct volume rendering of unstructured grids [S1]. It was developed to address the computational expense of repeated 3D inverse distance interpolation within tetrahedral cells [S1]. The method exploits the linear change of scalar values in any direction within a tetrahedral cell, a property inherent to the linear nature of the underlying scalar field representation in such cells [S1].

## Sources
[S1] Direct volume rendering of unstructured grids, https://doi.org/10.1016/s0097-8493(03)00034-7, fetched 2026-10-24
