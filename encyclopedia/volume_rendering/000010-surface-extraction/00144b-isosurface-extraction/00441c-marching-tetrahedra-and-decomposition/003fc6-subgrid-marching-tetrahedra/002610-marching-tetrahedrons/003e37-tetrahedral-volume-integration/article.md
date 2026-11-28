# Tetrahedral Volume Integration

Also called: Tetrahedral Volume Summation, Volume enclosed with the iso-surface (tetrahedral decomposition)

## Overview
Tetrahedral Volume Integration is a method for computing the scalar volume enclosed by an isosurface generated via tetrahedral decomposition [S1, unofficial]. It solves the problem of quantifying the material volume within a grid cell by summing the volumes of the tetrahedra parts that lie inside the isosurface [S1, unofficial]. Unlike surface extraction methods that only output geometry, this method provides a volumetric metric, distinguishing it from siblings like [Marching Tetrahedrons](3720-marching-tetrahedrons.md) which focus on triangle generation [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of tetrahedra from the decomposition of the grid cells, scalar field values at vertices, and an isovalue [S1, unofficial].
*   **Output:** The total enclosed volume $V_{total}$ [S1, unofficial].

**Data Structures:**
*   **Tetrahedron:** Defined by 4 vertices $\mathbf{v}_a, \mathbf{v}_b, \mathbf{v}_c, \mathbf{v}_d$ and their scalar values $s_a, s_b, s_c, s_d$ [S2, unofficial].
*   **Vertex State:** Binary classification of each vertex relative to the isovalue (Inside/On or Outside/Off) [S2, unofficial].

**Steps:**
1.  Iterate through each tetrahedron in the decomposition [S1, unofficial].
2.  Determine the position of the tetrahedron relative to the isosurface based on vertex states [S2, unofficial]:
    *   **Case 1 (All Outside):** If all vertices are 'off', the contribution is 0 [S2, unofficial].
    *   **Case 2 (All Inside):** If all vertices are 'on', the contribution is the full volume of the tetrahedron $V_{ol_t}$ [S2, unofficial].
    *   **Case 3 (Intersected):** If the tetrahedron is intersected by the isosurface, compute the volume of the part inside the isosurface [S1, unofficial]. This is done by identifying the intersection points on the edges and forming smaller tetrahedra with the original vertices [S1, unofficial].

**Formulas:**
The volume of a tetrahedron with vertices $\mathbf{p}_0, \mathbf{p}_1, \mathbf{p}_2, \mathbf{p}_3$ is [G]:
$$ V = \frac{1}{6} |(\mathbf{p}_1 - \mathbf{p}_0) \cdot ((\mathbf{p}_2 - \mathbf{p}_0) \times (\mathbf{p}_3 - \mathbf{p}_0))| $$

For intersected tetrahedra, the volume is decomposed into smaller tetrahedra formed by the intersection points and the original vertices [S1, unofficial]. The theoretical volume for validation is $V_r = \frac{4}{3} \pi r^3$, and the relative error is $E_r = \frac{V_d - V_r}{V_r}$ [S2, unofficial].

**Parameters:**
*   **Cell Edge Length:** The unit of measurement for object dimensions relative to the grid. Value: 1 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tetrahedron_volume(p0, p1, p2, p3):
    """Computes volume of a tetrahedron given 4 vertices."""
    return abs(dot(cross(p1 - p0, p2 - p0), p3 - p0)) / 6.0

def compute_enclosed_volume(vertices, scalars, isovalue):
    """
    Computes the volume of the part of a tetrahedron inside the isosurface.
    vertices: list of 4 vectors [va, vb, vc, vd]
    scalars: list of 4 floats [sa, sb, sc, sd]
    isovalue: float
    Returns: float volume
    """
    # Determine vertex states: 1 for inside/on, 0 for outside
    states = [1 if s <= isovalue else 0 for s in scalars]
    count_on = sum(states)
    
    # Group A: All same state
    if count_on == 0:
        return 0.0
    if count_on == 4:
        return tetrahedron_volume(*vertices)
    
    # Compute intersection points on all crossing edges
    intersections = []
    for i in range(4):
        for j in range(i + 1, 4):
            diff = scalars[j] - scalars[i]
            if abs(diff) < 1e-9:  # Guard against parallel edges [G]
                continue
            t = (isovalue - scalars[i]) / diff
            intersections.append(vertices[i] + t * (vertices[j] - vertices[i]))
                
    # Group B: 1 vertex different state
    if count_on == 1:
        v_t = vertices[states.index(1)]
        return tetrahedron_volume(v_t, *intersections)
    if count_on == 3:
        v_t = vertices[states.index(0)]
        vol_cut = tetrahedron_volume(v_t, *intersections)
        return tetrahedron_volume(*vertices) - vol_cut
        
    # Group C: 2 vertices different state
    if count_on == 2:
        on_indices = [i for i, s in enumerate(states) if s == 1]
        off_indices = [i for i, s in enumerate(states) if s == 0]
        va, vb = vertices[on_indices[0]], vertices[on_indices[1]]
        vc, vd = vertices[off_indices[0]], vertices[off_indices[1]]
        
        # Find intersection points on edges connecting on to off vertices
        t1 = None
        t2 = None
        for i in range(4):
            for j in range(i + 1, 4):
                diff = scalars[j] - scalars[i]
                if abs(diff) < 1e-9:
                    continue
                if states[i] != states[j]:
                    t = (isovalue - scalars[i]) / diff
                    p = vertices[i] + t * (vertices[j] - vertices[i])
                    if (i == on_indices[0] and j == off_indices[0]) or (i == off_indices[0] and j == on_indices[0]):
                        t1 = p
                    if (i == on_indices[1] and j == off_indices[1]) or (i == off_indices[1] and j == on_indices[1]):
                        t2 = p
                        
        # Fallback: decompose intersected polyhedron into two tetrahedra sharing edge (va, vb)
        if t1 is None or t2 is None:
            return tetrahedron_volume(va, vb, intersections[0], intersections[1])
            
        # Exact formula from [S2, unofficial]: Vol = Vol(a,b,c,t1) + Vol(a,b,d,t2) + Vol(a,b,t1,t2)
        return tetrahedron_volume(va, vb, vc, t1) + tetrahedron_volume(va, vb, vd, t2) + tetrahedron_volume(va, vb, t1, t2)
        
    return 0.0
```

## Subtopics

### Tetrahedral Volume Calculation via Vertex States
A method that calculates the volume $V_d$ defined by triangles inside tetrahedra based on the binary state (on/off) of the vertices [S2, unofficial]. It categorizes configurations into three groups:
*   **Group A:** All vertices same state [S2, unofficial]. Volume is $V_{ol_t}$ (if all on) or 0 (if all off) [S2, unofficial].
*   **Group B:** One vertex different state [S2, unofficial]. If one is on, volume is $V_{ol_{a,b,c,t}}$ (tetrahedron formed by the on vertex and the three triangle intersection points) [S2, unofficial]. If one is off, volume is $V_{ol_t} - V_{ol_{a,b,c,t}}$ [S2, unofficial].
*   **Group C:** Two vertices different state [S2, unofficial]. If two are on, volume is $\text{Vol}_{a,b,c,t1} + \text{Vol}_{a,b,d,t2} + \text{Vol}_{a,b,t1,t2}$ [S2, unofficial]. If two are off, volume is derived similarly [S2, unofficial].
This method provides a direct lookup or case-based calculation without explicit intersection point computation for every edge, relying on pre-defined geometric decompositions for each vertex state configuration [S2, unofficial].

### Exact Tetrahedral Surface Integration
A method for tetrahedral meshes that solves the surface integration exactly per tetrahedron [S3]. It was presented by Scheuermann et al. [S3].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of tetrahedra $N$, $O(N)$, as each tetrahedron is processed independently [S1, S2, unofficial].
*   **Memory Complexity:** $O(1)$ per tetrahedron, requiring only local vertex data [S1, S2, unofficial].
*   **Strengths:** Provides an exact volumetric measure for the isosurface, useful for quantitative analysis of scientific data [S1, S2, unofficial].
*   **Limitations:** Computationally more intensive than simple surface area calculation due to the need for geometric decomposition of intersected cells [S1, S2, unofficial].

## Background
The method originates from comparisons of fundamental isosurface extraction techniques, specifically focusing on the accuracy of volume and surface area measurements [S1, S2, unofficial]. It was developed to address the limitations of standard marching cubes/tetrahedra methods which do not inherently provide volumetric data [S1, S2, unofficial]. The vertex-state-based variant was introduced to optimize the calculation by leveraging pre-defined geometric configurations [S2, unofficial]. The exact surface integration approach was later formalized for tetrahedral meshes by Scheuermann et al. [S3].

## Sources
[S1] A Comparison of Fundamental Methods for Iso-surface Extraction, https://doi.org/10.48550/arxiv.2301.01715, fetched 2026-10-29 (unofficial)
[S2] Two Methods for Iso-Surface Extraction from Volumetric Data and Their Comparison, https://arxiv.org/abs/2201.03446v1, fetched 2026-10-26 (unofficial)
[S3] IRIS: Illustrative Rendering for Integral Surfaces, https://doi.org/10.1109/tvcg.2010.173, fetched 2026-10-27
